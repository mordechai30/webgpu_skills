# GPUWeb Specification

## 1. Introduction

*This section is non-normative.*

[Graphics Processing Units](https://en.wikipedia.org/wiki/Graphics_processing_unit), or GPUs for short,
have been essential in enabling rich rendering and computational applications in personal computing.
WebGPU is an API that exposes the capabilities of GPU hardware for the Web.
The API is designed from the ground up to efficiently map to (post-2014) native GPU APIs.
WebGPU is not related to [WebGL](https://www.khronos.org/webgl/) and does not explicitly target OpenGL ES.

WebGPU sees physical GPU hardware as [GPUAdapter](https://gpuweb.github.io#gpuadapter)[GPUDevice](https://gpuweb.github.io#gpudevice)[GPUQueue](https://gpuweb.github.io#gpuqueue)[GPUDevice](https://gpuweb.github.io#gpudevice)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUTexture](https://gpuweb.github.io#gputexture)[GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)[GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)[GPUShaderModule](https://gpuweb.github.io#gpushadermodule)[shader](https://gpuweb.github.io#shaders) code. The other resources,
such as [GPUSampler](https://gpuweb.github.io#gpusampler)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[physical resources](https://gpuweb.github.io#physical-resources) are used by the GPU.

GPUs execute commands encoded in [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)[pipeline](https://gpuweb.github.io#pipeline),
which is a mix of fixed-function and programmable stages. Programmable stages execute
shaders, which are special programs designed to run on GPU hardware.
Most of the state of a [pipeline](https://gpuweb.github.io#pipeline) is defined by
a [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[pipeline](https://gpuweb.github.io#pipeline) objects is set during encoding with commands,
such as [beginRenderPass()](https://gpuweb.github.io#dom-gpucommandencoder-beginrenderpass)[setBlendConstant()](https://gpuweb.github.io#dom-gpurenderpassencoder-setblendconstant)

## 2. Malicious use considerations

*This section is non-normative.* It describes the risks associated with exposing this API on the Web.

### 2.1. Security Considerations

The security requirements for WebGPU are the same as ever for the web, and are likewise non-negotiable. The general approach is strictly validating all the commands before they reach GPU, ensuring that a page can only work with its own data.

#### 2.1.1. CPU-based undefined behavior

A WebGPU implementation translates the workloads issued by the user into API commands specific
to the target platform. Native APIs specify the valid usage for the commands
(for example, see [vkCreateDescriptorSetLayout](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateDescriptorSetLayout.html))
and generally don’t guarantee any outcome if the valid usage rules are not followed.
This is called "undefined behavior", and it can be exploited by an attacker to access memory
they don’t own, or force the driver to execute arbitrary code.

In order to disallow insecure usage, the range of allowed WebGPU behaviors is defined for any input.
An implementation has to validate all the input from the user and only reach the driver
with the valid workloads. This document specifies all the error conditions and handling semantics.
For example, specifying the same buffer with intersecting ranges in both "source" and "destination"
of [copyBufferToBuffer()](https://gpuweb.github.io#gpucommandencoder-copybuffertobuffer) results in [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)

See [§ 22 Errors & Debugging](https://gpuweb.github.io#errors-and-debugging) for more information about error handling.

#### 2.1.2. GPU-based undefined behavior

WebGPU [shader](https://gpuweb.github.io#shaders)s are executed by the compute units inside GPU hardware. In native APIs,
some of the shader instructions may result in undefined behavior on the GPU.
In order to address that, the shader instruction set and its defined behaviors are
strictly defined by WebGPU. When a shader is provided to [createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)

#### 2.1.3. Uninitialized data

Generally, allocating new memory may expose the leftover data of other applications running on the system. In order to address that, WebGPU conceptually initializes all the resources to zero, although in practice an implementation may skip this step if it sees the developer initializing the contents manually. This includes variables and shared workgroup memory inside shaders.

The precise mechanism of clearing the workgroup memory can differ between platforms. If the native API does not provide facilities to clear it, the WebGPU implementation transforms the compute shader to first do a clear across all invocations, synchronize them, and continue executing developer’s code.

[GPULoadOp](https://gpuweb.github.io#enumdef-gpuloadop)["load"](https://gpuweb.github.io#dom-gpuloadop-load)["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
As a result, all implementations **should** issue a developer console warning about this
    potential performance penalty, even if there is no penalty in that implementation.

#### 2.1.4. Out-of-bounds access in shaders

[Shader](https://gpuweb.github.io#shaders)s can access [physical resource](https://gpuweb.github.io#physical-resources)s either directly
(for example, as a ["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform)[GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)[texture unit](https://gpuweb.github.io#texture-unit)s are not involved.

In order to prevent the shaders from accessing GPU memory an application doesn’t own, the WebGPU implementation may enable a special mode (called "robust buffer access") in the driver that guarantees that the access is limited to buffer bounds.

Alternatively, an implementation may transform the shader code by inserting manual bounds checks.
When this path is taken, the out-of-bound checks only apply to array indexing. They aren’t needed
for plain field access of shader structures due to the [minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)

If the shader attempts to load data outside of [physical resource](https://gpuweb.github.io#physical-resources) bounds,
the implementation is allowed to:

1. return a value at a different location within the resource bounds
2. return a value vector of "(0, 0, 0, X)" with any "X"
3. partially discard the draw or dispatch call

If the shader attempts to write data outside of [physical resource](https://gpuweb.github.io#physical-resources) bounds,
the implementation is allowed to:

1. write the value to a different location within the resource bounds
2. discard the write operation
3. partially discard the draw or dispatch call

#### 2.1.5. Invalid data

When uploading [floating-point](https://en.wikipedia.org/wiki/IEEE_754) data from CPU to GPU,
or generating it on the GPU, we may end up with a binary representation that doesn’t correspond
to a valid number, such as infinity or NaN (not-a-number). The GPU behavior in this case is
subject to the accuracy of the GPU hardware implementation of the IEEE-754 standard.
WebGPU guarantees that introducing invalid floating-point numbers would only affect the results
of arithmetic computations and will not have other side effects.

#### 2.1.6. Driver bugs

GPU drivers are subject to bugs like any other software. If a bug occurs, an attacker could possibly exploit the incorrect behavior of the driver to get access to unprivileged data. In order to reduce the risk, the WebGPU working group will coordinate with GPU vendors to integrate the WebGPU Conformance Test Suite (CTS) as part of their driver testing process, like it was done for WebGL. WebGPU implementations are expected to have workarounds for some of the discovered bugs, and disable WebGPU on drivers with known bugs that can’t be worked around.

#### 2.1.7. Timing attacks

##### 2.1.7.1. Content-timeline timing

WebGPU does not expose new states to JavaScript (the [content timeline](https://gpuweb.github.io#content-timeline)) which are
shared between [agents](https://tc39.es/ecma262/#agent) in an [agent cluster](https://tc39.es/ecma262/#sec-agent-clusters).
[Content timeline](https://gpuweb.github.io#content-timeline) states such as [\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[content timeline](https://gpuweb.github.io#content-timeline) tasks, like in plain JavaScript.

##### 2.1.7.2. Device/queue-timeline timing

Writable storage buffers and other cross-invocation communication may be usable to construct
high-precision timers on the [queue timeline](https://gpuweb.github.io#queue-timeline).

The optional ["timestamp-query"](https://gpuweb.github.io#timestamp-query)[current queue timestamp](https://gpuweb.github.io#abstract-opdef-current-queue-timestamp). Note in particular:

- The [device timeline](https://gpuweb.github.io#device-timeline) typically runs in a process that is shared by multiple
origins, so cross-origin isolation (provided by COOP/COEP) does not provide
isolation of device/queue-timeline timers.
- [Queue timeline](https://gpuweb.github.io#queue-timeline) work is issued from the device timeline, and may execute on GPU hardware that
does not provide the isolation expected of CPU processes (such as Meltdown mitigations).
- GPU hardware is not typically susceptible to Spectre-style attacks, **but** WebGPU may be
implemented in software, and software implementations may run in a shared process, preventing
isolation-based mitigations.

#### 2.1.8. Row hammer attacks

[Row hammer](https://en.wikipedia.org/wiki/Row_hammer) is a class of attacks that exploit the
leaking of states in DRAM cells. It could be used [on GPU](https://www.vusec.net/projects/glitch/).
WebGPU does not have any specific mitigations in place, and relies on platform-level solutions,
such as reduced memory refresh intervals.

#### 2.1.9. Denial of service

WebGPU applications have access to GPU memory and compute units. A WebGPU implementation may limit the available GPU memory to an application, in order to keep other applications responsive. For GPU processing time, a WebGPU implementation may set up "watchdog" timer that makes sure an application doesn’t cause GPU unresponsiveness for more than a few seconds. These measures are similar to those used in WebGL.

#### 2.1.10. Workload identification

WebGPU provides access to constrained global resources shared between different programs (and web pages) running on the same machine. An application can try to indirectly probe how constrained these global resources are, in order to reason about workloads performed by other open web pages, based on the patterns of usage of these shared resources. These issues are generally analogous to issues with Javascript, such as system memory and CPU execution throughput. WebGPU does not provide any additional mitigations for this.

#### 2.1.11. Memory resources

WebGPU exposes fallible allocations from machine-global memory heaps, such as VRAM. This allows for probing the size of the system’s remaining available memory (for a given heap type) by attempting to allocate and watching for allocation failures.

GPUs internally have one or more (typically only two) heaps of memory shared by all running applications. When a heap is depleted, WebGPU would fail to create a resource. This is observable, which may allow a malicious application to guess what heaps are used by other applications, and how much they allocate from them.

#### 2.1.12. Computation resources

If one site uses WebGPU at the same time as another, it may observe the increase in time it takes to process some work. For example, if a site constantly submits compute workloads and tracks completion of work on the queue, it may observe that something else also started using the GPU.

A GPU has many parts that can be tested independently, such as the arithmetic units, texture sampling units, atomic units, etc. A malicious application may sense when some of these units are stressed, and attempt to guess the workload of another application by analyzing the stress patterns. This is analogous to the realities of CPU execution of Javascript.

#### 2.1.13. Abuse of capabilities

Malicious sites could abuse the capabilities exposed by WebGPU to run computations that don’t benefit the user or their experience and instead only benefit the site. Examples would be hidden crypto-mining, password cracking or rainbow tables computations.

It is not possible to guard against these types of uses of the API because the browser is not able to distinguish between valid workloads and abusive workloads. This is a general problem with all general-purpose computation capabilities on the Web: JavaScript, WebAssembly or WebGL. WebGPU only makes some workloads easier to implement, or slightly more efficient to run than using WebGL.

To mitigate this form of abuse, browsers can throttle operations on background tabs, could warn that a tab is using a lot of resource, and restrict which contexts are allowed to use WebGPU.

User agents can heuristically issue warnings to users about high power use, especially due to potentially malicious usage. If a user agent implements such a warning, it should include WebGPU usage in its heuristics, in addition to JavaScript, WebAssembly, WebGL, and so on.

### 2.2. Privacy Considerations

 
[The privacy considerations for WebGPU are similar to those of WebGL. GPU APIs are complex and must
expose various aspects of a device’s capabilities out of necessity in order to enable developers to
take advantage of those capabilities effectively. The general mitigation approach involves
normalizing or binning potentially identifying information and enforcing uniform behavior where
possible.](https://infra.spec.whatwg.org/#tracking-vector)

A user agent must not reveal more than 32 distinguishable configurations or buckets.

#### 2.2.1. Machine-specific features and limits

WebGPU can expose a lot of detail on the underlying GPU architecture and the device geometry. This includes available physical adapters, many limits on the GPU and CPU resources that could be used (such as the maximum texture size), and any optional hardware-specific capabilities that are available.

User agents are not obligated to expose the real hardware limits, they are in full control of how much the machine specifics are exposed. One strategy to reduce fingerprinting is binning all the target platforms into a few number of bins. In general, the privacy impact of exposing the hardware limits matches the one of WebGL.

The [default](https://gpuweb.github.io#limit-default) limits are also deliberately high enough
to allow most applications to work without requesting higher limits.
All the usage of the API is validated according to the requested limits,
so the actual hardware capabilities are not exposed to the users by accident.

#### 2.2.2. Machine-specific artifacts

There are some machine-specific rasterization/precision artifacts and performance differences that can be observed roughly in the same way as in WebGL. This applies to rasterization coverage and patterns, interpolation precision of the varyings between shader stages, compute unit scheduling, and more aspects of execution.

Generally, rasterization and precision fingerprints are identical across most or all of the devices of each vendor. Performance differences are relatively intractable, but also relatively low-signal (as with JS execution performance).

Privacy-critical applications and user agents should utilize software implementations to eliminate such artifacts.

#### 2.2.3. Machine-specific performance

Another factor for differentiating users is measuring the performance of specific operations on the GPU. Even with low precision timing, repeated execution of an operation can show if the user’s machine is fast at specific workloads. This is a fairly common vector (present in both WebGL and Javascript), but it’s also low-signal and relatively intractable to truly normalize.

WebGPU compute pipelines expose access to GPU unobstructed by the fixed-function hardware. This poses an additional risk for unique device fingerprinting. User agents can take steps to dissociate logical GPU invocations with actual compute units to reduce this risk.

#### 2.2.4. User Agent State

This specification doesn’t define any additional user-agent state for an origin.
However it is expected that user agents will have compilation caches for the result of expensive
compilation like [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)

For the specification, these caches are indifferentiable from incredibly fast compilation, but
for applications it would be easy to measure how long [createComputePipelineAsync()](https://gpuweb.github.io#dom-gpudevice-createcomputepipelineasync)[storage partitioning](https://github.com/privacycg/storage-partitioning).

The system’s GPU driver may also have its own cache of compiled shaders and pipelines. User agents may want to disable these when at all possible, or add per-partition data to shaders in ways that will make the GPU driver consider them different.

#### 2.2.5. Driver bugs

In addition to the concerns outlined in [Security Considerations](https://gpuweb.github.io#security-driver-bugs), driver
bugs may introduce differences in behavior that can be observed as a method of differentiating
users. The mitigations mentioned in Security Considerations apply here as well, including
coordinating with GPU vendors and implementing workarounds for known issues in the user agent.

#### 2.2.6. Adapter Identifiers

Past experience with WebGL has demonstrated that developers have a legitimate need to be able to identify the GPU their code is running on in order to create and maintain robust GPU-based content. For example, to identify adapters with known driver bugs in order to work around them or to avoid features that perform more poorly than expected on a given class of hardware.

But exposing adapter identifiers also naturally expands the amount of fingerprinting information available, so there’s a desire to limit the precision with which we identify the adapter.

There are several mitigations that can be applied to strike a balance between enabling robust content and preserving privacy. First is that user agents can reduce the burden on developers by identifying and working around known driver issues, as they have since browsers began making use of GPUs.

When adapter identifiers are exposed by default they should be as broad as possible while still being useful. Possibly identifying, for example, the adapter’s vendor and general architecture without identifying the specific adapter in use. Similarly, in some cases identifiers for an adapter that is considered a reasonable proxy for the actual adapter may be reported.

In cases where full and detailed information about the adapter is useful (for example: when filing bug reports) the user can be asked for consent to reveal additional information about their hardware to the page.

Finally, the user agent will always have the discretion to not report adapter identifiers at all if it considers it appropriate, such as in enhanced privacy modes.

## 3. Fundamentals

### 3.1. Conventions

#### 3.1.1. Syntactic Shorthands

In this specification, the following syntactic shorthands are used:

- The `.` ("dot") syntax, common in programming languages.
- The phrasing "`Foo.Bar`" means "the `Bar` member of the value (or interface) `Foo`."
If `Foo` is an [ordered map](https://infra.spec.whatwg.org/#ordered-map) and `Bar` does not [exist](https://infra.spec.whatwg.org/#map-exists) in `Foo`, returns `undefined`.
The phrasing "`Foo.Bar` is [provided](https://infra.spec.whatwg.org/#map-exists)" means
"the `Bar` member [exists](https://infra.spec.whatwg.org/#map-exists) in the [map](https://infra.spec.whatwg.org/#ordered-map) value `Foo`"
- The `?.` ("optional chaining") syntax, adopted from JavaScript.
- The phrasing "`Foo?.Bar`" means
"if `Foo` is `null` or `undefined` or `Bar` does not [exist](https://infra.spec.whatwg.org/#map-exists) in `Foo`, `undefined`; otherwise, `Foo.Bar`".
For example, where `buffer` is a [GPUBuffer](https://gpuweb.github.io#gpubuffer)`buffer?.\[[device]].\[[adapter]]` means
"if `buffer` is `null` or `undefined`, then `undefined`; otherwise,
the `\[[adapter]]` internal slot of the `\[[device]]` internal slot of `buffer`.
- The `??` ("nullish coalescing") syntax, adopted from JavaScript.
- The phrasing "`x` ?? `y`" means "`x`, if `x` is not null or undefined, and `y` otherwise".
- slot-backed attribute
- A WebIDL attribute which is backed by an internal slot of the same name. It may or may not be mutable.

#### 3.1.2. WebGPU Objects

A WebGPU object consists of a [WebGPU Interface](https://gpuweb.github.io#webgpu-interface) and an [internal object](https://gpuweb.github.io#internal-object).

The WebGPU interface defines the public interface and state of the [WebGPU object](https://gpuweb.github.io#webgpu-object).
It can be used on the [content timeline](https://gpuweb.github.io#content-timeline) where it was created, where it is a JavaScript-exposed
WebIDL interface.

Any interface which includes `GPUObjectBase` is a [WebGPU interface](https://gpuweb.github.io#webgpu-interface).

The internal object tracks the state of the [WebGPU object](https://gpuweb.github.io#webgpu-object) on the [device timeline](https://gpuweb.github.io#device-timeline).
All reads/writes to the mutable state of an [internal object](https://gpuweb.github.io#internal-object) occur from steps executing on a
single well-ordered [device timeline](https://gpuweb.github.io#device-timeline).

The following special property types can be defined on [WebGPU objects](https://gpuweb.github.io#webgpu-object):

- immutable property
- A read-only slot set during initialization of the object. It can be accessed from any timeline. Note: Since the slot is immutable, implementations may have a copy on multiple timelines, as needed.
[Immutable properties](https://gpuweb.github.io#immutable-property) are defined in this way to avoid describing multiple copies in this spec.
If named `[[with brackets]]`, it is an internal slot.
If named `withoutBrackets`, it is a `readonly` [slot-backed attribute](https://gpuweb.github.io#slot-backed-attribute) of the [WebGPU interface](https://gpuweb.github.io#webgpu-interface).
- content timeline property
- A property which is only accessible from the [content timeline](https://gpuweb.github.io#content-timeline) where the object was created.
If named `[[with brackets]]`, it is an internal slot.
If named `withoutBrackets`, it is a [slot-backed attribute](https://gpuweb.github.io#slot-backed-attribute) of the [WebGPU interface](https://gpuweb.github.io#webgpu-interface).
- device timeline property
- A property which tracks state of the [internal object](https://gpuweb.github.io#internal-object) and is only accessible from the [device timeline](https://gpuweb.github.io#device-timeline) where the object was created. [device timeline properties](https://gpuweb.github.io#device-timeline-property) may be mutable.
[Device timeline properties](https://gpuweb.github.io#device-timeline-property) are named `[[with brackets]]`, and are internal slots.
- queue timeline property
- A property which tracks state of the [internal object](https://gpuweb.github.io#internal-object) and is only accessible from the [queue timeline](https://gpuweb.github.io#queue-timeline) where the object was created. [queue timeline properties](https://gpuweb.github.io#queue-timeline-property) may be mutable.
[Queue timeline properties](https://gpuweb.github.io#queue-timeline-property) are named `[[with brackets]]`, and are internal slots.

interface mixin GPUObjectBase attribute USVString label 

[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)
`parent`, interface

`T`,

[GPUObjectDescriptorBase](https://gpuweb.github.io#dictdef-gpuobjectdescriptorbase)
`descriptor`) (where

`T`extends

[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)
[content timeline](https://gpuweb.github.io#content-timeline)steps:

1. Let `device` be `parent`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
2. Let `object` be a new instance of `T`.
3. Set `object`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)`device`.
4. Set `object`.[label](https://gpuweb.github.io#dom-gpuobjectbase-label)`descriptor`.[label](https://gpuweb.github.io#dom-gpuobjectdescriptorbase-label)
5. Return `object`.

[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)[immutable properties](https://gpuweb.github.io#immutable-property):

- `[[device]]`, of type [device](https://gpuweb.github.io#device), readonly
- The [device](https://gpuweb.github.io#device) that owns the [internal object](https://gpuweb.github.io#internal-object).
Operations on the contents of this object [assert](https://infra.spec.whatwg.org/#assert) they are running on the [device timeline](https://gpuweb.github.io#device-timeline), and that the device is [valid](https://gpuweb.github.io#abstract-opdef-valid).

[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)[content timeline properties](https://gpuweb.github.io#content-timeline-property):

- `label`,  of type [USVString](https://webidl.spec.whatwg.org/#idl-USVString)
- A developer-provided label which is used in an [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined) way.
It can be used by the browser, OS, or other tools to help
identify the underlying [internal object](https://gpuweb.github.io#internal-object) to the developer.
Examples include displaying the label in [GPUError](https://gpuweb.github.io#gpuerror)
NOTE:Implementations**should** use labels to enhance error messages by using them to
    identify WebGPU objects. However, this need not be the only way of identifying objects:
    implementations **should** also use other available information,
    especially when no label is available. For example:
  - The label of the parent [GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)
  - The label of the parent [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)
  - The label of the source [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)
  - The label of the source [GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)[GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)

  NOTE:The[label](https://gpuweb.github.io#dom-gpuobjectbase-label)
[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)
[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)
[getBindGroupLayout()](https://gpuweb.github.io#dom-gpupipelinebase-getbindgrouplayout)
[label](https://gpuweb.github.io#dom-gpuobjectbase-label)
This means one underlying object could be associated with multiple labels.
    This specification does not define how the label is propagated to the [device timeline](https://gpuweb.github.io#device-timeline).
    How labels are used is completely [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined): error messages could
    show the most recently set label, all known labels, or no labels at all.
It is defined as a [USVString](https://webidl.spec.whatwg.org/#idl-USVString)

[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[valid]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
`true`.
- If `true`, indicates that the [internal object](https://gpuweb.github.io#internal-object) is valid to use.

[WebGPU interfaces](https://gpuweb.github.io#webgpu-interface)should not prevent their parent objects, such as the

[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
As a result, developers should assume that a [WebGPU interface](https://gpuweb.github.io#webgpu-interface) may remain live
    until all child objects of that interface have also been garbage collected, causing some
    resources to remain allocated longer than anticipated.

Calling the `destroy` method on a [WebGPU interface](https://gpuweb.github.io#webgpu-interface) (such as
    [GPUDevice](https://gpuweb.github.io#gpudevice)[destroy()](https://gpuweb.github.io#dom-gpudevice-destroy)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[destroy()](https://gpuweb.github.io#dom-gpubuffer-destroy)

#### 3.1.3. Object Descriptors

An object descriptor holds the information needed to create an object,
which is typically done via one of the `create*` methods of [GPUDevice](https://gpuweb.github.io#gpudevice)

dictionary GPUObjectDescriptorBase USVString label 

[GPUObjectDescriptorBase](https://gpuweb.github.io#dictdef-gpuobjectdescriptorbase)

- `label`,  of type [USVString](https://webidl.spec.whatwg.org/#idl-USVString), defaulting to `""`
- The initial value of [GPUObjectBase.label](https://gpuweb.github.io#dom-gpuobjectbase-label)

### 3.2. Asynchrony

#### 3.2.1. Invalid Internal Objects & Contagious Invalidity

Object creation operations in WebGPU don’t return promises, but nonetheless are internally
asynchronous. Returned objects refer to [internal objects](https://gpuweb.github.io#internal-object) which are manipulated on a
[device timeline](https://gpuweb.github.io#device-timeline). Rather than fail with exceptions or rejections, most errors that occur on a
[device timeline](https://gpuweb.github.io#device-timeline) are communicated through [GPUError](https://gpuweb.github.io#gpuerror)[device](https://gpuweb.github.io#device).

[Internal objects](https://gpuweb.github.io#internal-object) are either [valid](https://gpuweb.github.io#abstract-opdef-valid) or [invalid](https://gpuweb.github.io#abstract-opdef-invalid).
An [invalid](https://gpuweb.github.io#abstract-opdef-invalid) object will never become [valid](https://gpuweb.github.io#abstract-opdef-valid) at a later time,
but some [valid](https://gpuweb.github.io#abstract-opdef-valid) objects may be [invalidated](https://gpuweb.github.io#abstract-opdef-invalidate).

Objects are [invalid](https://gpuweb.github.io#abstract-opdef-invalid) from creation if it wasn’t possible to create them.
This can happen, for example, if the [object descriptor](https://gpuweb.github.io#object-descriptor) doesn’t describe a valid
object, or if there is not enough memory to allocate a resource.
It can also happen if an object is created with or from another invalid object
(for example calling [createView()](https://gpuweb.github.io#dom-gputexture-createview)[GPUTexture](https://gpuweb.github.io#gputexture)[GPUTexture](https://gpuweb.github.io#gputexture)[createView()](https://gpuweb.github.io#dom-gputexture-createview)

[Internal objects](https://gpuweb.github.io#internal-object) of *most* types cannot become [invalid](https://gpuweb.github.io#abstract-opdef-invalid) after they are created, but still
may become unusable, e.g. if the owning device is [lost](https://gpuweb.github.io#lose-the-device) or
[destroyed](https://gpuweb.github.io#dom-gpudevice-destroy)[destroyed](https://gpuweb.github.io#gpubuffer-internal-state-destroyed)".

[Internal objects](https://gpuweb.github.io#internal-object) of some types *can* become [invalid](https://gpuweb.github.io#abstract-opdef-invalid) after they are created; specifically,
[devices](https://gpuweb.github.io#device), [adapters](https://gpuweb.github.io#adapter), [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)

[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)
`object`is valid to use with a

`targetObject`if the all of the requirements in the following

[device timeline](https://gpuweb.github.io#device-timeline)steps are met:

- `object`.[\[\[valid\]\]](https://gpuweb.github.io#dom-gpuobjectbase-valid-slot)`true`.
- `object`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[valid\]\]](https://gpuweb.github.io#dom-gpuobjectbase-valid-slot)`true`.
- `object`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)`targetObject`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)

[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)
`object`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. `object`.[\[\[valid\]\]](https://gpuweb.github.io#dom-gpuobjectbase-valid-slot)`false`.

#### 3.2.2. Promise Ordering

Several operations in WebGPU return promises.

WebGPU does not make any guarantees about the order in which these promises settle (resolve or reject), except for the following:

- 
     For some[GPUQueue](https://gpuweb.github.io#gpuqueue)
`q`,
    if `p1` = `q`.[onSubmittedWorkDone()](https://gpuweb.github.io#dom-gpuqueue-onsubmittedworkdone)
`p2` = `q`.[onSubmittedWorkDone()](https://gpuweb.github.io#dom-gpuqueue-onsubmittedworkdone)
`p1` must settle before `p2`.
- 
     For some[GPUQueue](https://gpuweb.github.io#gpuqueue)
`q` and [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`b` on the same [GPUDevice](https://gpuweb.github.io#gpudevice)
`p1` = `b`.[mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)
`p2` = `q`.[onSubmittedWorkDone()](https://gpuweb.github.io#dom-gpuqueue-onsubmittedworkdone)
`p1` must settle before `p2`.

Applications must not rely on any other promise settlement ordering.

### 3.3. Coordinate Systems

Rendering operations use the following coordinate systems:

- Normalized device coordinates (or NDC) have three dimensions, where: 
  - -1.0 ≤ x ≤ 1.0
  - -1.0 ≤ y ≤ 1.0
  - 0.0 ≤ z ≤ 1.0
  - The bottom-left corner is at (-1.0, -1.0, z).
Normalized device coordinates. Note: Whether `z = 0` or `z = 1` is treated as the near plane is application specific. The above diagram presents `z = 0` as the near plane but the observed behavior is determined by a combination of the projection matrices
used by shaders, the [depthClearValue](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthclearvalue)[depthCompare](https://gpuweb.github.io#dom-gpudepthstencilstate-depthcompare)
- Clip space coordinates have four dimensions: (x, y, z, w) 
  - Clip space coordinates are used for the the [clip position](https://gpuweb.github.io#clip-position) of a vertex (i.e. the [position](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-position) output of a vertex shader),
and for the [clip volume](https://gpuweb.github.io#clip-volume).
  - [Normalized device coordinates](https://gpuweb.github.io#ndc) and clip space coordinates are related as follows:
If point *p = (p.x, p.y, p.z, p.w)* is in the [clip volume](https://gpuweb.github.io#clip-volume), then its normalized device coordinates are (*p.x* ÷ *p.w*, *p.y* ÷ *p.w*, *p.z* ÷ *p.w*).
- Framebuffer coordinates address the pixels in the [framebuffer](https://gpuweb.github.io#framebuffer)
  - They have two dimensions.
  - Each pixel extends 1 unit in x and y dimensions.
  - The top-left corner is at (0.0, 0.0).
  - x increases to the right.
  - y increases down.
  - See [§ 17 Render Passes](https://gpuweb.github.io#render-passes) and [§ 23.2.5 Rasterization](https://gpuweb.github.io#rasterization).
Framebuffer coordinates.
- Viewport coordinates combine [framebuffer coordinates](https://gpuweb.github.io#framebuffer-coordinates) in x and y dimensions,
with depth in z.
  - Normally 0.0 ≤ z ≤ 1.0, but this can be modified by setting [\[\[viewport\]\]](https://gpuweb.github.io#dom-renderstate-viewport-slot)`minDepth` and `maxDepth` via [setViewport()](https://gpuweb.github.io#dom-gpurenderpassencoder-setviewport)
- Fragment coordinates match [viewport coordinates](https://gpuweb.github.io#viewport-coordinates).
- Texture coordinates, sometimes called "UV coordinates" in 2D, are used to sample
textures and have a number of components matching the [texture dimension](https://gpuweb.github.io#enumdef-gputexturedimension)
  - 0 ≤ u ≤ 1.0
  - 0 ≤ v ≤ 1.0
  - 0 ≤ w ≤ 1.0
  - (0.0, 0.0, 0.0) is in the first texel in texture memory address order.
  - (1.0, 1.0, 1.0) is in the last texel texture memory address order.
2D Texture coordinates.
- Window coordinates, or present coordinates,
match [framebuffer coordinates](https://gpuweb.github.io#framebuffer-coordinates), and are used when interacting with
an external display or conceptually similar interface.

Note: WebGPU’s coordinate systems match DirectX’s coordinate systems in a graphics pipeline.

### 3.4. Programming Model

#### 3.4.1. Timelines

WebGPU’s behavior is described in terms of "timelines". Each operation (defined as algorithms) occurs on a timeline. Timelines clearly define both the order of operations, and which state is available to which operations.

Note:
This "timeline" model describes the constraints of the multi-process models of
browser engines (typically with a "content process" and "GPU process"), as well
as the GPU itself as a separate execution unit in many implementations.
Implementing WebGPU does not require timelines to execute in parallel, so does
not require multiple processes, or even multiple threads.
(It does require concurrency for cases like [get a copy of the image contents of a context](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context)
which synchronously blocks on another timeline to complete.)

- Content timeline
- Associated with the execution of the Web script. It includes calling all methods described by this specification. To issue steps to the content timeline from an operation on [GPUDevice](https://gpuweb.github.io#gpudevice)`device`, [queue a global task for GPUDevice](https://gpuweb.github.io#abstract-opdef-queue-a-global-task-for-gpudevice) `device` with those steps.
- Device timeline
- Associated with the GPU device operations that are issued by the user agent. It includes creation of adapters, devices, and GPU resources and state objects, which are typically synchronous operations from the point of view of the user agent part that controls the GPU, but can live in a separate OS process.
- Queue timeline
- Associated with the execution of operations on the compute units of the GPU. It includes actual draw, copy, and compute jobs that run on the GPU.
- Timeline-agnostic
- Associated with any of the above timelines Steps may be issued to any timeline if they only operate on [immutable properties](https://gpuweb.github.io#immutable-property) or
arguments passed from the calling steps.

[The following show the styling of steps and values associated with each timeline. This styling is non-normative; the specification text always describes the association.](https://gpuweb.github.io#example-05808e3e)

- Immutable value example term definition
- Can be used on any timeline.

- Content-timeline example term definition
- Can only be used on the [content timeline](https://gpuweb.github.io#content-timeline).

- Device-timeline example term definition
- Can only be used on the [device timeline](https://gpuweb.github.io#device-timeline).

- Queue-timeline example term definition
- Can only be used on the [queue timeline](https://gpuweb.github.io#queue-timeline).

[timeline-agnostic](https://gpuweb.github.io#timeline-agnostic)look like this.

[Immutable value example term](https://gpuweb.github.io#immutable-value-example-term) usage.

[content timeline](https://gpuweb.github.io#content-timeline)look like this.

[Immutable value example term](https://gpuweb.github.io#immutable-value-example-term) usage.
            [Content-timeline example term](https://gpuweb.github.io#content-timeline-example-term) usage.

[device timeline](https://gpuweb.github.io#device-timeline)look like this.

[Immutable value example term](https://gpuweb.github.io#immutable-value-example-term) usage.
            [Device-timeline example term](https://gpuweb.github.io#device-timeline-example-term) usage.

[queue timeline](https://gpuweb.github.io#queue-timeline)look like this.

[Immutable value example term](https://gpuweb.github.io#immutable-value-example-term) usage.
            [Queue-timeline example term](https://gpuweb.github.io#queue-timeline-example-term) usage.

In this specification, asynchronous operations are used when the return value
depends on work that happens on any timeline other than the [Content timeline](https://gpuweb.github.io#content-timeline).
They are represented by promises and events in API.


[GPUComputePassEncoder.dispatchWorkgroups()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups)
1. User encodes a `dispatchWorkgroups` command by calling a method of the [GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)[Content timeline](https://gpuweb.github.io#content-timeline).
2. User issues [GPUQueue.submit()](https://gpuweb.github.io#dom-gpuqueue-submit)[GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)[Device timeline](https://gpuweb.github.io#device-timeline) by calling the OS driver to do a low-level submission.
3. The submit gets dispatched by the GPU invocation scheduler onto the
actual compute units for execution, which happens on the [Queue timeline](https://gpuweb.github.io#queue-timeline).


[GPUDevice.createBuffer()](https://gpuweb.github.io#dom-gpudevice-createbuffer)
1. User fills out a [GPUBufferDescriptor](https://gpuweb.github.io#gpubufferdescriptor)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[Content timeline](https://gpuweb.github.io#content-timeline).
2. User agent creates a low-level buffer on the [Device timeline](https://gpuweb.github.io#device-timeline).


[GPUBuffer.mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)
1. User requests to map a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[Content timeline](https://gpuweb.github.io#content-timeline) and
gets a promise in return.
2. User agent checks if the buffer is currently used by the GPU and makes a reminder to itself to check back when this usage is over.
3. After the GPU operating on [Queue timeline](https://gpuweb.github.io#queue-timeline) is done using the buffer,
the user agent maps it to memory and [resolves](https://webidl.spec.whatwg.org/#resolve) the promise.

#### 3.4.2. Memory Model

*This section is non-normative.*

Once a [GPUDevice](https://gpuweb.github.io#gpudevice)

1. User agent implementing the specification.
2. Operating system with low-level native API drivers for this device.
3. Actual CPU and GPU hardware.

Each layer of the [WebGPU platform](https://gpuweb.github.io#webgpu-platform) may have different memory types
that the user agent needs to consider when implementing the specification:

- The script-owned memory, such as an [ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)
- A user agent may have different processes responsible for running the content and communication to the GPU driver. In this case, it uses inter-process shared memory to transfer data.
- Dedicated GPUs have their own memory with high bandwidth, while integrated GPUs typically share memory with the system.

Most [physical resources](https://gpuweb.github.io#physical-resources) are allocated in the memory of type
that is efficient for computation or rendering by the GPU.
When the user needs to provide new data to the GPU,
the data may first need to cross the process boundary in order to reach
the user agent part that communicates with the GPU driver.
Then it may need to be made visible to the driver,
which sometimes requires a copy into driver-allocated staging memory.
Finally, it may need to be transferred to the dedicated GPU memory,
potentially changing the internal layout into one
that is most efficient for GPUs to operate on.

All of these transitions are done by the WebGPU implementation of the user agent.

Note: This example describes the worst case, while in practice
the implementation might not need to cross the process boundary,
or may be able to expose the driver-managed memory directly to
the user behind an `ArrayBuffer`, thus avoiding any data copies.

#### 3.4.3. Resource Usages

A [physical resource](https://gpuweb.github.io#physical-resources) can be used with an internal usage by a [GPU command](https://gpuweb.github.io#gpu-command):

- input
- Buffer with input data for draw or dispatch calls. Preserves the contents.
Allowed by buffer [INDEX](https://gpuweb.github.io#dom-gpubufferusage-index)[VERTEX](https://gpuweb.github.io#dom-gpubufferusage-vertex)[INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)
- constant
- Resource bindings that are constant from the shader point of view. Preserves the contents.
Allowed by buffer [UNIFORM](https://gpuweb.github.io#dom-gpubufferusage-uniform)[TEXTURE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-texture_binding)
- storage
- Read/write storage resource binding.
Allowed by buffer [STORAGE](https://gpuweb.github.io#dom-gpubufferusage-storage)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)
- storage-read
- Read-only storage resource bindings. Preserves the contents.
Allowed by buffer [STORAGE](https://gpuweb.github.io#dom-gpubufferusage-storage)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)
- attachment
- Texture used as a read/write output attachment or
write-only resolve target in a render pass.
Allowed by texture [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)
- attachment-read
- Texture used as a read-only attachment in a render pass. Preserves the contents.
Allowed by texture [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)

We define subresource to be either a whole buffer, or a [texture subresource](https://gpuweb.github.io#texture-subresources).

[internal usages](https://gpuweb.github.io#internal-usage)are compatible with others. A

[subresource](https://gpuweb.github.io#subresource)can be in a state that combines multiple usages together. We consider a list

`U`to be a compatible usage list if (and only if) it satisfies any of the following rules:

- Each usage in `U` is [input](https://gpuweb.github.io#internal-usage-input), [constant](https://gpuweb.github.io#internal-usage-constant), [storage-read](https://gpuweb.github.io#internal-usage-storage-read), or [attachment-read](https://gpuweb.github.io#internal-usage-attachment-read).
- Each usage in `U` is [storage](https://gpuweb.github.io#internal-usage-storage).
Multiple such usages are allowed even though they are writable.
This is the [usage scope storage exception](https://gpuweb.github.io#usage-scope-storage-exception).
- Each usage in `U` is [attachment](https://gpuweb.github.io#internal-usage-attachment).
Multiple such usages are allowed even though they are writable.
This is the [usage scope attachment exception](https://gpuweb.github.io#usage-scope-attachment-exception).

Enforcing that the usages are only combined into a [compatible usage list](https://gpuweb.github.io#compatible-usage-list)
allows the API to limit when data races can occur in working with memory.
That property makes applications written against
WebGPU more likely to run without modification on different platforms.


[storage](https://gpuweb.github.io#internal-usage-storage)as well as for

[input](https://gpuweb.github.io#internal-usage-input)within the same

[GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
[compatible usage list](https://gpuweb.github.io#compatible-usage-list)for that buffer.


- [attachment-read](https://gpuweb.github.io#internal-usage-attachment-read)
As a depth/stencil attachment with all aspects marked read-only
(using [depthReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthreadonly)[stencilReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilreadonly)
- [constant](https://gpuweb.github.io#internal-usage-constant)
As a texture binding to a draw call.


- A buffer or texture may be bound as [storage](https://gpuweb.github.io#internal-usage-storage) to two
different draw calls in a render pass.
- Disjoint ranges of a single buffer may be bound to two different binding
points as [storage](https://gpuweb.github.io#internal-usage-storage).
Overlapping ranges must not be bound to a single dispatch/draw call;
this is checked by "[Encoder bind groups alias a writable resource](https://gpuweb.github.io#abstract-opdef-encoder-bind-groups-alias-a-writable-resource)".


[attachment](https://gpuweb.github.io#internal-usage-attachment)more than once. This is necessary to allow disjoint slices of 3D textures to be bound as different attachments to a single render pass.

One slice must not be bound twice for two different attachments;
    this is checked by [beginRenderPass()](https://gpuweb.github.io#dom-gpucommandencoder-beginrenderpass)

#### 3.4.4. Synchronization

A usage scope is a [map](https://infra.spec.whatwg.org/#ordered-map) from [subresource](https://gpuweb.github.io#subresource) to [list](https://infra.spec.whatwg.org/#list)\<[internal usage](https://gpuweb.github.io#internal-usage)>>.
Each usage scope covers a range of operations which may execute in a concurrent
fashion with each other, and therefore may only use [subresources](https://gpuweb.github.io#subresource) in consistent
    [compatible usage lists](https://gpuweb.github.io#compatible-usage-list) within the scope.

[usage scope](https://gpuweb.github.io#usage-scope)

`scope`passes usage scope validation if, for each \[

`subresource`, `usageList`\] in

`scope`,

`usageList`is a

[compatible usage list](https://gpuweb.github.io#compatible-usage-list).

[subresource](https://gpuweb.github.io#subresource)

`subresource`to

[usage scope](https://gpuweb.github.io#usage-scope)

`usageScope`with usage (

[internal usage](https://gpuweb.github.io#internal-usage)or set of

[internal usages](https://gpuweb.github.io#internal-usage))

`usage`:

[usage scope](https://gpuweb.github.io#usage-scope)

`A`into

[usage scope](https://gpuweb.github.io#usage-scope)

`B`:

1. For each \[`subresource`, `usage`\] in `A`:
  1. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) `subresource` to `B` with usage `usage`.

[Usage scopes](https://gpuweb.github.io#usage-scope) are constructed and validated during encoding:

- in [dispatchWorkgroups()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups)
- in [dispatchWorkgroupsIndirect()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroupsindirect)
- at [GPURenderPassEncoder.end()](https://gpuweb.github.io#dom-gpurenderpassencoder-end)
- at [GPURenderBundleEncoder.finish()](https://gpuweb.github.io#dom-gpurenderbundleencoder-finish)

The [usage scopes](https://gpuweb.github.io#usage-scope) are as follows:

- In a compute pass, each dispatch command ([dispatchWorkgroups()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups)[dispatchWorkgroupsIndirect()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroupsindirect)
A subresource is used in the usage scope if it is potentially accessible by the dispatched invocations, including: 
  - All [subresources](https://gpuweb.github.io#subresource) referenced by bind groups in slots used by the current [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[\[\[layout\]\]](https://gpuweb.github.io#dom-gpupipelinebase-layout-slot)
  - Buffers used directly by dispatch calls (such as indirect buffers)
Note:
State-setting compute pass commands, like
[setBindGroup()](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup),
do not contribute their bound resources directly to a usage scope: they only change the
state that is checked in dispatch commands.
- One render pass is one usage scope. A subresource is used in the usage scope if it’s referenced by any command, including state-setting commands (unlike in compute passes), including: 
  - Buffers set by [setVertexBuffer()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setvertexbuffer)
  - Buffers set by [setIndexBuffer()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setindexbuffer)
  - All [subresources](https://gpuweb.github.io#subresource) referenced by bind groups set by [setBindGroup()](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup)
  - Buffers used directly by draw calls (such as indirect buffers)

Note: Copy commands are standalone operations and don’t use [usage scopes](https://gpuweb.github.io#usage-scope) for validation.
They implement their own validation to prevent self-races.


*are*included in

[usage scopes](https://gpuweb.github.io#usage-scope):

- In a render pass, subresources used in any
[setBindGroup()](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup) call, regardless of whether the currently bound pipeline’s
shader or layout actually depends on these bindings,
or the bind group is shadowed by another 'set' call.
- A buffer used in any [setVertexBuffer()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setvertexbuffer)
- A buffer used in any [setIndexBuffer()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setindexbuffer)
- A texture subresource used as a color attachment, resolve attachment, or
depth/stencil attachment in [GPURenderPassDescriptor](https://gpuweb.github.io#dictdef-gpurenderpassdescriptor)[beginRenderPass()](https://gpuweb.github.io#dom-gpucommandencoder-beginrenderpass)
- Resources used in bind group entries with visibility 0, or visible only to the compute stage but used in a render pass (or vice versa).

### 3.5. Core Internal Objects

#### 3.5.1. Adapters

An adapter identifies an implementation of WebGPU on the system: both an instance of compute/rendering functionality on the platform underlying a browser, and an instance of a browser’s implementation of WebGPU on top of that functionality.

[Adapters](https://gpuweb.github.io#adapter) are exposed via [GPUAdapter](https://gpuweb.github.io#gpuadapter)

[Adapters](https://gpuweb.github.io#adapter) do not uniquely represent underlying implementations:
calling [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)[adapter](https://gpuweb.github.io#adapter)
object each time.

Each [adapter](https://gpuweb.github.io#adapter) object can only be used to create one [device](https://gpuweb.github.io#device):
upon a successful [requestDevice()](https://gpuweb.github.io#dom-gpuadapter-requestdevice)[\[\[state\]\]](https://gpuweb.github.io#dom-adapter-state-slot)["consumed"](https://gpuweb.github.io#dom-adapter-state-consumed)[adapter](https://gpuweb.github.io#adapter) objects may [expire](https://gpuweb.github.io#abstract-opdef-expire) at any time.

Note:
This ensures applications use the latest system state for adapter selection when creating a device.
It also encourages robustness to more scenarios by making them look similar: first initialization,
reinitialization due to an unplugged adapter, reinitialization due to a test
[GPUDevice.destroy()](https://gpuweb.github.io#dom-gpudevice-destroy)

An [adapter](https://gpuweb.github.io#adapter) may be considered a fallback adapter if it has significant performance
caveats in exchange for some combination of wider compatibility, more predictable behavior, or
improved privacy. It is not required that a [fallback adapter](https://gpuweb.github.io#fallback-adapter) is available on every system.

[adapter](https://gpuweb.github.io#adapter) has the following [immutable properties](https://gpuweb.github.io#immutable-property):

- `[[features]]`, of type [ordered set](https://infra.spec.whatwg.org/#ordered-set)\<[GPUFeatureName](https://gpuweb.github.io#gpufeaturename)
- The [features](https://gpuweb.github.io#feature) which can be used to create devices on this adapter.
- `[[limits]]`, of type [supported limits](https://gpuweb.github.io#supported-limits), readonly
- The [best](https://gpuweb.github.io#limit-better) limits which can be used to create devices on this adapter.
Each adapter limit must be the same or [better](https://gpuweb.github.io#limit-better) than its default value
in [supported limits](https://gpuweb.github.io#supported-limits).
- `[[fallback]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- If set to `true` indicates that the adapter is a [fallback adapter](https://gpuweb.github.io#fallback-adapter).
- `[[xrCompatible]]`, of type boolean
- If set to `true` indicates that the adapter was requested with compatibility with [WebXR sessions](https://www.w3.org/TR/webxr/#session).
- `[[default feature level]]`, of type [feature level string](https://gpuweb.github.io#feature-level-string), readonly
- Indicates the default feature level of devices created from this adapter.

[adapter](https://gpuweb.github.io#adapter) has the following [device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[state]]`, initially ["valid"](https://gpuweb.github.io#dom-adapter-state-valid)
  - `"valid"`
  - The adapter can be used to create a device.
  - `"consumed"`
  - The adapter has already been used to create a device, and cannot be used again.
  - `"expired"`
  - The adapter has expired for some other reason.

[GPUAdapter](https://gpuweb.github.io#gpuadapter)
`adapter`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. Set `adapter`.[\[\[adapter\]\]](https://gpuweb.github.io#dom-gpuadapter-adapter-slot)[\[\[state\]\]](https://gpuweb.github.io#dom-adapter-state-slot)["expired"](https://gpuweb.github.io#dom-adapter-state-expired)

#### 3.5.2. Devices

A device is the logical instantiation of an [adapter](https://gpuweb.github.io#adapter),
through which [internal objects](https://gpuweb.github.io#internal-object) are created.

[Devices](https://gpuweb.github.io#device) are exposed via [GPUDevice](https://gpuweb.github.io#gpudevice)

A [device](https://gpuweb.github.io#device) is the exclusive owner of all [internal objects](https://gpuweb.github.io#internal-object) created from it:
when the [device](https://gpuweb.github.io#device) becomes [invalid](https://gpuweb.github.io#abstract-opdef-invalid)
(is [lost](https://gpuweb.github.io#lose-the-device) or [destroyed](https://gpuweb.github.io#dom-gpudevice-destroy)[createTexture()](https://gpuweb.github.io#dom-gpudevice-createtexture)[createView()](https://gpuweb.github.io#dom-gputexture-createview)[unusable](https://gpuweb.github.io#abstract-opdef-valid-to-use-with).

[device](https://gpuweb.github.io#device) has the following [immutable properties](https://gpuweb.github.io#immutable-property):

- `[[adapter]]`, of type [adapter](https://gpuweb.github.io#adapter), readonly
- The [adapter](https://gpuweb.github.io#adapter) from which this device was created.
- `[[features]]`, of type [ordered set](https://infra.spec.whatwg.org/#ordered-set)\<[GPUFeatureName](https://gpuweb.github.io#gpufeaturename)
- The [features](https://gpuweb.github.io#feature) which can be used on this device, as computed [at creation](https://gpuweb.github.io#a-new-device).
No additional features can be used, even if the underlying [adapter](https://gpuweb.github.io#adapter) can support them.
- `[[limits]]`, of type [supported limits](https://gpuweb.github.io#supported-limits), readonly
- The limits which can be used on this device, as computed [at creation](https://gpuweb.github.io#a-new-device).
No [better](https://gpuweb.github.io#limit-better) limits can be used, even if the underlying [adapter](https://gpuweb.github.io#adapter) can support them.

[device](https://gpuweb.github.io#device) has the following [content timeline properties](https://gpuweb.github.io#content-timeline-property):

- `[[content device]]`, of type [GPUDevice](https://gpuweb.github.io#gpudevice)
- The [Content timeline](https://gpuweb.github.io#content-timeline) [GPUDevice](https://gpuweb.github.io#gpudevice)

[adapter](https://gpuweb.github.io#adapter)

`adapter`with

[GPUDeviceDescriptor](https://gpuweb.github.io#gpudevicedescriptor)
`descriptor`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. Let `features` be the [set](https://infra.spec.whatwg.org/#ordered-set) of values in `descriptor`.[requiredFeatures](https://gpuweb.github.io#dom-gpudevicedescriptor-requiredfeatures)
2. [Extend](https://infra.spec.whatwg.org/#set-extend) `features` with any default [GPUFeatureName](https://gpuweb.github.io#gpufeaturename)`adapter`.[\[\[default feature level\]\]](https://gpuweb.github.io#dom-adapter-default-feature-level-slot)
3. For each `feature` in `features`:
  1. [Extend](https://infra.spec.whatwg.org/#set-extend) `features` with the set of features required (directly or indirectly)
by `feature`, as defined in [§ 3.6.1.2 Feature Dependencies](https://gpuweb.github.io#feature-dependencies).
4. Let `limits` be a new [supported limits](https://gpuweb.github.io#supported-limits) object with the default limits
as defined by the `adapter`.[\[\[default feature level\]\]](https://gpuweb.github.io#dom-adapter-default-feature-level-slot)
5. For each (`key`, `value`) pair in `descriptor`.[requiredLimits](https://gpuweb.github.io#dom-gpudevicedescriptor-requiredlimits)
  1. If `value` is not `undefined` and `value` is [better](https://gpuweb.github.io#limit-better) than `limits`\[`key`\]:
    1. Set `limits`\[`key`\] to `value`.
6. Set `limits`.[maxStorageBuffersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebufferspershaderstage)`limits`.[maxStorageBuffersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebufferspershaderstage)`limits`.[maxStorageBuffersInVertexStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebuffersinvertexstage)`limits`.[maxStorageBuffersInFragmentStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebuffersinfragmentstage)
7. Set `limits`.[maxStorageTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturespershaderstage)`limits`.[maxStorageTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturespershaderstage)`limits`.[maxStorageTexturesInVertexStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturesinvertexstage)`limits`.[maxStorageTexturesInFragmentStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturesinfragmentstage)
8. If `features` [contains](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
  1. Set `limits`.[maxStorageBuffersInVertexStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebuffersinvertexstage)`limits`.[maxStorageBuffersInFragmentStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebuffersinfragmentstage)`limits`.[maxStorageBuffersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebufferspershaderstage)
  2. Set `limits`.[maxStorageTexturesInVertexStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturesinvertexstage)`limits`.[maxStorageTexturesInFragmentStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturesinfragmentstage)`limits`.[maxStorageTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturespershaderstage)
9. Let `device` be a [device](https://gpuweb.github.io#device) object.
10. Set `device`.[\[\[adapter\]\]](https://gpuweb.github.io#dom-device-adapter-slot)`adapter`.
11. Set `device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)`features`.
12. Set `device`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)`limits`.
13. Return `device`.

Any time the user agent needs to revoke access to a device, it calls
[lose the device](https://gpuweb.github.io#lose-the-device)(`device`, ["unknown"](https://gpuweb.github.io#dom-gpudevicelostreason-unknown)[device timeline](https://gpuweb.github.io#device-timeline),
potentially ahead of other operations currently queued on that timeline.

If an operation fails with side effects that would observably change the state
of objects on the device or potentially corrupt internal implementation/driver state,
the device **should** be lost to prevent these changes from being observable.

Note:
For all device losses not initiated by the application (via [destroy()](https://gpuweb.github.io#dom-gpudevice-destroy)*unconditionally*,
even if the [lost](https://gpuweb.github.io#dom-gpudevice-lost)

`device`,

`reason`) run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. [Invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `device`.
2. Issue the following steps on the [content timeline](https://gpuweb.github.io#content-timeline) of `device`.[\[\[content device\]\]](https://gpuweb.github.io#dom-device-content-device-slot)
  1. Resolve `device`.[lost](https://gpuweb.github.io#dom-gpudevice-lost)[GPUDeviceLostInfo](https://gpuweb.github.io#gpudevicelostinfo)[reason](https://gpuweb.github.io#dom-gpudevicelostinfo-reason)`reason` and [message](https://gpuweb.github.io#dom-gpudevicelostinfo-message)[implementation-defined](https://infra.spec.whatwg.org/#implementation-defined) value.
Note: [message](https://gpuweb.github.io#dom-gpudevicelostinfo-message)
3. Complete any outstanding steps that are waiting until `device` becomes lost.

Note: No errors are generated from a device which is lost.
    See [§ 22 Errors & Debugging](https://gpuweb.github.io#errors-and-debugging).

`event`on

[device](https://gpuweb.github.io#device)

`device`, handled by

`steps`on timeline

`timeline`:

- If or when the [device timeline](https://gpuweb.github.io#device-timeline) has been informed of the completion of `event`, or
- If `device` is [lost](https://gpuweb.github.io#abstract-opdef-invalid) already, or when it [becomes lost](https://gpuweb.github.io#becomes-lost):

Then issue `steps` on `timeline`.

### 3.6. Optional Capabilities

WebGPU [adapters](https://gpuweb.github.io#adapter) and [devices](https://gpuweb.github.io#device) have capabilities, which
describe WebGPU functionality that differs between different implementations,
typically due to hardware or system software constraints.
A [capability](https://gpuweb.github.io#capabilities) is either a [feature](https://gpuweb.github.io#feature) or a [limit](https://gpuweb.github.io#limit).

A user agent must not reveal more than 32 distinguishable configurations or buckets.

The capabilities of an [adapter](https://gpuweb.github.io#adapter) must conform to [§ 4.2.1 Adapter Capability Guarantees](https://gpuweb.github.io#adapter-capability-guarantees).

Only supported capabilities may be requested in [requestDevice()](https://gpuweb.github.io#dom-gpuadapter-requestdevice)

The capabilities of a [device](https://gpuweb.github.io#device) are determined in "[a new device](https://gpuweb.github.io#a-new-device)" by starting with the adapter’s
defaults (no features and the default [supported limits](https://gpuweb.github.io#supported-limits))
and adding capabilities as requested in [requestDevice()](https://gpuweb.github.io#dom-gpuadapter-requestdevice)[adapter](https://gpuweb.github.io#adapter).

 
[For privacy considerations, see](https://infra.spec.whatwg.org/#tracking-vector) [§ 2.2.1 Machine-specific features and limits](https://gpuweb.github.io#privacy-machine-limits).

#### 3.6.1. Features

A feature is a set of optional WebGPU functionality that is not supported on all implementations, typically due to hardware or system software constraints.

All [features](https://gpuweb.github.io#feature) are optional, but [adapters](https://gpuweb.github.io#adapter) make some guarantees about their availability
(see [§ 4.2.1 Adapter Capability Guarantees](https://gpuweb.github.io#adapter-capability-guarantees)).

A [device](https://gpuweb.github.io#device) supports the exact set of features determined at creation (see [§ 3.6 Optional Capabilities](https://gpuweb.github.io#optional-capabilities)).
API calls perform validation according to the device’s features (not the [adapter](https://gpuweb.github.io#adapter)’s features):

- Using existing API surfaces in a non-enabled way **typically** results in a [validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).
- Using a non-enabled method or enum value always throws a [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)
- Using a non-enabled dictionary member with a (correctly-typed) non-default value **typically** results in a [validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).
- Using a non-enabled WGSL `enable` directive always results in a [createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)[validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).

[GPUFeatureName](https://gpuweb.github.io#gpufeaturename)
`feature`is enabled for a

[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)
`object`if and only if

`object`.

[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contains](https://infra.spec.whatwg.org/#list-contain)

`feature`.

See the [Feature Index](https://gpuweb.github.io#feature-index) for a description of the functionality each feature enables.

Note: Even where supported, enabling features is not necessarily desirable, as doing so may have a performance impact. Because of this, and to improve portability across devices and implementations, applications should generally only request features that they may actually require.

##### 3.6.1.1. `GPUFeatureName`

Each [GPUFeatureName](https://gpuweb.github.io#gpufeaturename)

enum GPUFeatureName "atomic-vec2u-min-max" "core-features-and-limits" "depth-clip-control" "depth32float-stencil8" "texture-compression-bc" "texture-compression-bc-sliced-3d" "texture-compression-etc2" "texture-compression-astc" "texture-compression-astc-sliced-3d" "timestamp-query" "indirect-first-instance" "shader-f16" "rg11b10ufloat-renderable" "bgra8unorm-storage" "float32-filterable" "float32-blendable" "clip-distances" "dual-source-blending" "subgroups" "texture-formats-tier1" "texture-formats-tier2" "primitive-index" "texture-component-swizzle" "subgroup-size-control" "texture-compression-unaligned" 

##### 3.6.1.2. Feature Dependencies

A more-advanced feature `F2` may require a less-advanced feature `F1`.

- In [§ 4.2.1 Adapter Capability Guarantees](https://gpuweb.github.io#adapter-capability-guarantees), adapters with `F2` are guaranteed to also have `F1`.
- In [a new device](https://gpuweb.github.io#a-new-device), devices created with `F2` will also implicitly enable `F1`.

| This feature... | requires these features and their dependencies. | 
|---|---|
| ["texture-formats-tier2"](https://gpuweb.github.io#texture-formats-tier2) |  | 
| ["texture-formats-tier1"](https://gpuweb.github.io#texture-formats-tier1) |  | 
| ["texture-compression-bc-sliced-3d"](https://gpuweb.github.io#texture-compression-bc-sliced-3d) |  | 
| ["texture-compression-astc-sliced-3d"](https://gpuweb.github.io#texture-compression-astc-sliced-3d) |  | 
| ["subgroup-size-control"](https://gpuweb.github.io#dom-gpufeaturename-subgroup-size-control) |  | 

#### 3.6.2. Limits

Each limit is a numeric limit on the usage of WebGPU on a device.

Note: Even where supported, setting "better" limits is not necessarily desirable, as doing so may have a performance impact. Because of this, and to improve portability across devices and implementations, applications should generally only request limits better than the defaults if they may actually require them.

Each limit has a default value and a compatibility mode default.

[Adapters](https://gpuweb.github.io#adapter) are always guaranteed to support the defaults or [better](https://gpuweb.github.io#limit-better)
(see [§ 4.2.1 Adapter Capability Guarantees](https://gpuweb.github.io#adapter-capability-guarantees)).

A [device](https://gpuweb.github.io#device) supports the exact set of limits determined at creation (see [§ 3.6 Optional Capabilities](https://gpuweb.github.io#optional-capabilities)).
API calls perform validation according to these limits (not the [adapter](https://gpuweb.github.io#adapter)’s limits),
no [better](https://gpuweb.github.io#limit-better) or worse.

For any given limit, some values are better than others.
A [better](https://gpuweb.github.io#limit-better) limit value always relaxes validation, enabling strictly
more programs to be valid. For each [limit class](https://gpuweb.github.io#limit-class), "better" is defined.

Different limits have different limit classes:

- maximum
- The limit enforces a maximum on some value passed into the API. Higher values are [better](https://gpuweb.github.io#limit-better).
May only be set to values ≥ the [default](https://gpuweb.github.io#limit-default).
Lower values are clamped to the [default](https://gpuweb.github.io#limit-default).
- alignment
- The limit enforces a minimum alignment on some value passed into the API; that is, the value must be a multiple of the limit. Lower values are [better](https://gpuweb.github.io#limit-better).
May only be set to powers of 2 which are ≤ the [default](https://gpuweb.github.io#limit-default).
Values which are not powers of 2 are invalid.
Higher powers of 2 are clamped to the [default](https://gpuweb.github.io#limit-default).

A supported limits object has a value for every limit defined by WebGPU:

| Limit name | Type | [Limit class](https://gpuweb.github.io#limit-class) | [Default](https://gpuweb.github.io#limit-default) | [Compatibility Mode Default](https://gpuweb.github.io#limit-compatibility-mode-default) | 
|---|---|---|---|---|
| `maxTextureDimension1D` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 8192 | 4096 | 
| The maximum allowed value for the [size](https://gpuweb.github.io#dom-gputexturedescriptor-size).[width](https://gpuweb.github.io#gpuextent3d-width) of a [texture](https://gpuweb.github.io#texture) created with [dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension) ["1d"](https://gpuweb.github.io#dom-gputexturedimension-1d). |  |  |  |  | 
| `maxTextureDimension2D` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 8192 | 4096 | 
| The maximum allowed value for the [size](https://gpuweb.github.io#dom-gputexturedescriptor-size).[width](https://gpuweb.github.io#gpuextent3d-width) and [size](https://gpuweb.github.io#dom-gputexturedescriptor-size).[height](https://gpuweb.github.io#gpuextent3d-height) of a [texture](https://gpuweb.github.io#texture) created with [dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension) ["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d). |  |  |  |  | 
| `maxTextureDimension3D` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 2048 |  | 
| The maximum allowed value for the [size](https://gpuweb.github.io#dom-gputexturedescriptor-size).[width](https://gpuweb.github.io#gpuextent3d-width), [size](https://gpuweb.github.io#dom-gputexturedescriptor-size).[height](https://gpuweb.github.io#gpuextent3d-height) and [size](https://gpuweb.github.io#dom-gputexturedescriptor-size).[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) of a [texture](https://gpuweb.github.io#texture) created with [dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension) ["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d). |  |  |  |  | 
| `maxTextureArrayLayers` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 256 |  | 
| The maximum allowed value for the [size](https://gpuweb.github.io#dom-gputexturedescriptor-size).[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) of a [texture](https://gpuweb.github.io#texture) created with [dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension) ["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d). |  |  |  |  | 
| `maxBindGroups` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 4 |  | 
| The maximum number of [GPUBindGroupLayouts](https://gpuweb.github.io#gpubindgrouplayout) allowed in [bindGroupLayouts](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-bindgrouplayouts) when creating a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout). |  |  |  |  | 
| `maxBindGroupsPlusVertexBuffers` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 24 |  | 
| The maximum number of bind group and vertex buffer slots used simultaneously,         counting any empty slots below the highest index.         Validated in [createRenderPipeline()](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline) and [in draw calls](https://gpuweb.github.io#abstract-opdef-valid-to-draw). |  |  |  |  | 
| `maxImmediateSize` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 64 |  | 
| The maximum size, in bytes, of immediate data range used in a pipeline. Note: 64 bytes is the size of a 4×4 matrix of f32 values. |  |  |  |  | 
| `maxBindingsPerBindGroup` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 1000 |  | 
| The number of binding indices available when creating a [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout). Note: This limit is normative, but arbitrary.         With the default [binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits), it is impossible         to use 1000 bindings in one bind group, but this allows .[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) values up to 999.         This limit allows implementations to treat binding space as an array,         within reasonable memory space, rather than a sparse map structure.[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding) |  |  |  |  | 
| `maxDynamicUniformBuffersPerPipelineLayout` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 8 |  | 
| The maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are uniform buffers with dynamic offsets.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxDynamicStorageBuffersPerPipelineLayout` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 4 |  | 
| The maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are storage buffers with dynamic offsets.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxSampledTexturesPerShaderStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 16 |  | 
| For each possible [GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage) `stage`,         the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are sampled textures.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxSamplersPerShaderStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 16 |  | 
| For each possible [GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage) `stage`,         the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are samplers.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxStorageBuffersPerShaderStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 8 |  | 
| For each possible [GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage) `stage`,         the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are storage buffers.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). Note: This limit applies to all stages. At [device initialization](https://gpuweb.github.io#a-new-device), it is normalized with and [maxStorageBuffersInVertexStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebuffersinvertexstage) so that in the validation algorithm, each stage can be checked against just one of the three limits.[maxStorageBuffersInFragmentStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebuffersinfragmentstage) |  |  |  |  | 
| `maxStorageBuffersInVertexStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 8 | 0 | 
| For the vertex stage, the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are storage buffers.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxStorageBuffersInFragmentStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 8 | 4 | 
| For the fragment stage, the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are storage buffers.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxStorageTexturesPerShaderStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 4 |  | 
| For each possible [GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage) `stage`,         the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are storage textures.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). Note: This limit applies to all stages. At [device initialization](https://gpuweb.github.io#a-new-device), it is normalized with and [maxStorageTexturesInVertexStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturesinvertexstage) so that in the validation algorithm, each stage can be checked against just one of the three limits.[maxStorageTexturesInFragmentStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturesinfragmentstage) |  |  |  |  | 
| `maxStorageTexturesInVertexStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 4 | 0 | 
| For the vertex stage, the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are storage textures.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxStorageTexturesInFragmentStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 4 |  | 
| For the fragment stage, the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are storage textures.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxUniformBuffersPerShaderStage` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 12 |  | 
| For each possible [GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage) `stage`,         the maximum number of [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) entries across a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout) which are uniform buffers.         See [Exceeds the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits). |  |  |  |  | 
| `maxUniformBufferBindingSize` | [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 65536 bytes | 16384 bytes | 
| The maximum [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding).[size](https://gpuweb.github.io#dom-gpubufferbinding-size) for bindings with a [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) `entry` for which `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)?.[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type) is ["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform). |  |  |  |  | 
| `maxStorageBufferBindingSize` | [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 134217728 bytes (128 MiB) |  | 
| The maximum [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding).[size](https://gpuweb.github.io#dom-gpubufferbinding-size) for bindings with a [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) `entry` for which `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)?.[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type) is ["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage) or ["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage). |  |  |  |  | 
| `minUniformBufferOffsetAlignment` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [alignment](https://gpuweb.github.io#limit-class-alignment) | 256 bytes |  | 
| The required alignment for [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding).[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset) and         the dynamic offsets provided in [setBindGroup()](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup),         for bindings with a [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) `entry` for which `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)?.[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type) is ["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform). |  |  |  |  | 
| `minStorageBufferOffsetAlignment` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [alignment](https://gpuweb.github.io#limit-class-alignment) | 256 bytes |  | 
| The required alignment for [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding).[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset) and         the dynamic offsets provided in [setBindGroup()](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup),         for bindings with a [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry) `entry` for which `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)?.[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type) is ["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage) or ["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage). |  |  |  |  | 
| `maxVertexBuffers` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 8 |  | 
| The maximum number of [buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers) when creating a [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline). |  |  |  |  | 
| `maxBufferSize` | [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 268435456 bytes (256 MiB) |  | 
| The maximum size of [size](https://gpuweb.github.io#dom-gpubufferdescriptor-size) when creating a [GPUBuffer](https://gpuweb.github.io#gpubuffer). |  |  |  |  | 
| `maxVertexAttributes` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 16 |  | 
| The maximum number of [attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes) in total across [buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers) when creating a [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline). |  |  |  |  | 
| `maxVertexBufferArrayStride` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 2048 bytes |  | 
| The maximum allowed [arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride) when creating a [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline). |  |  |  |  | 
| `maxInterStageShaderVariables` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 16 | 15 | 
| The maximum allowed number of input or output variables for inter-stage communication (like vertex outputs or fragment inputs). |  |  |  |  | 
| `maxColorAttachments` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 8 | 4 | 
| The maximum allowed number of color attachments in [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor).[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment).[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets), [GPURenderPassDescriptor](https://gpuweb.github.io#dictdef-gpurenderpassdescriptor).[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments),         and [GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout).[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats). |  |  |  |  | 
| `maxColorAttachmentBytesPerSample` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 32 |  | 
| The maximum number of bytes necessary to hold one sample (pixel or subpixel) of render pipeline output data, across all color attachments. |  |  |  |  | 
| `maxComputeWorkgroupStorageSize` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 16384 bytes |  | 
| The maximum number of bytes of [workgroup](https://gpuweb.github.io/gpuweb/wgsl/#address-spaces-workgroup) storage used for a compute stage [GPUShaderModule](https://gpuweb.github.io#gpushadermodule) entry-point. |  |  |  |  | 
| `maxComputeInvocationsPerWorkgroup` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 256 | 128 | 
| The maximum value of the product of the `workgroup_size` dimensions for a         compute stage [GPUShaderModule](https://gpuweb.github.io#gpushadermodule) entry-point. |  |  |  |  | 
| `maxComputeWorkgroupSizeX` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 256 | 128 | 
| The maximum value of the `workgroup_size` X dimension for a         compute stage [GPUShaderModule](https://gpuweb.github.io#gpushadermodule) entry-point. |  |  |  |  | 
| `maxComputeWorkgroupSizeY` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 256 | 128 | 
| The maximum value of the `workgroup_size` Y dimensions for a         compute stage [GPUShaderModule](https://gpuweb.github.io#gpushadermodule) entry-point. |  |  |  |  | 
| `maxComputeWorkgroupSizeZ` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 64 |  | 
| The maximum value of the `workgroup_size` Z dimensions for a         compute stage [GPUShaderModule](https://gpuweb.github.io#gpushadermodule) entry-point. |  |  |  |  | 
| `maxComputeWorkgroupsPerDimension` | [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32) | [maximum](https://gpuweb.github.io#limit-class-maximum) | 65535 |  | 
| The maximum value for the arguments of [dispatchWorkgroups(workgroupCountX, workgroupCountY, workgroupCountZ)](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups). |  |  |  |  | 

##### 3.6.2.1. `GPUSupportedLimits`

[GPUSupportedLimits](https://gpuweb.github.io#gpusupportedlimits)[supported limits](https://gpuweb.github.io#supported-limits).
See [GPUAdapter.limits](https://gpuweb.github.io#dom-gpuadapter-limits)[GPUDevice.limits](https://gpuweb.github.io#dom-gpudevice-limits)

\[Exposed Window ,Worker ),SecureContext interface GPUSupportedLimits readonly attribute unsigned long maxTextureDimension1D readonly attribute unsigned long maxTextureDimension2D readonly attribute unsigned long maxTextureDimension3D readonly attribute unsigned long maxTextureArrayLayers readonly attribute unsigned long maxBindGroups readonly attribute unsigned long maxBindGroupsPlusVertexBuffers readonly attribute unsigned long maxImmediateSize readonly attribute unsigned long maxBindingsPerBindGroup readonly attribute unsigned long maxDynamicUniformBuffersPerPipelineLayout readonly attribute unsigned long maxDynamicStorageBuffersPerPipelineLayout readonly attribute unsigned long maxSampledTexturesPerShaderStage readonly attribute unsigned long maxSamplersPerShaderStage readonly attribute unsigned long maxStorageBuffersPerShaderStage readonly attribute unsigned long maxStorageBuffersInVertexStage readonly attribute unsigned long maxStorageBuffersInFragmentStage readonly attribute unsigned long maxStorageTexturesPerShaderStage readonly attribute unsigned long maxStorageTexturesInVertexStage readonly attribute unsigned long maxStorageTexturesInFragmentStage readonly attribute unsigned long maxUniformBuffersPerShaderStage readonly attribute unsigned long long maxUniformBufferBindingSize readonly attribute unsigned long long maxStorageBufferBindingSize readonly attribute unsigned long minUniformBufferOffsetAlignment readonly attribute unsigned long minStorageBufferOffsetAlignment readonly attribute unsigned long maxVertexBuffers readonly attribute unsigned long long maxBufferSize readonly attribute unsigned long maxVertexAttributes readonly attribute unsigned long maxVertexBufferArrayStride readonly attribute unsigned long maxInterStageShaderVariables readonly attribute unsigned long maxColorAttachments readonly attribute unsigned long maxColorAttachmentBytesPerSample readonly attribute unsigned long maxComputeWorkgroupStorageSize readonly attribute unsigned long maxComputeInvocationsPerWorkgroup readonly attribute unsigned long maxComputeWorkgroupSizeX readonly attribute unsigned long maxComputeWorkgroupSizeY readonly attribute unsigned long maxComputeWorkgroupSizeZ readonly attribute unsigned long maxComputeWorkgroupsPerDimension 

##### 3.6.2.2. `GPUSupportedFeatures`

[GPUSupportedFeatures](https://gpuweb.github.io#gpusupportedfeatures)[setlike](https://webidl.spec.whatwg.org/#dfn-setlike) interface. Its [set entries](https://webidl.spec.whatwg.org/#dfn-set-entries) are
the [GPUFeatureName](https://gpuweb.github.io#gpufeaturename)[features](https://gpuweb.github.io#feature) supported by an adapter or
device. It must only contain strings from the [GPUFeatureName](https://gpuweb.github.io#gpufeaturename)

\[Exposed Window ,Worker ),SecureContext interface GPUSupportedFeatures readonly setlike DOMString 

[GPUSupportedFeatures](https://gpuweb.github.io#gpusupportedfeatures)
[set entries](https://webidl.spec.whatwg.org/#dfn-set-entries)is

[DOMString](https://webidl.spec.whatwg.org/#idl-DOMString)[GPUFeatureName](https://gpuweb.github.io#gpufeaturename)
[set entries](https://webidl.spec.whatwg.org/#dfn-set-entries)type was

[GPUFeatureName](https://gpuweb.github.io#gpufeaturename)[TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)`false`:
    
    
   ##### 3.6.2.3. `WGSLLanguageFeatures`

[WGSLLanguageFeatures](https://gpuweb.github.io#gpuwgsllanguagefeatures)[setlike](https://webidl.spec.whatwg.org/#dfn-setlike) interface of
`navigator.gpu.`.
Its [wgslLanguageFeatures](https://gpuweb.github.io#dom-gpu-wgsllanguagefeatures)[set entries](https://webidl.spec.whatwg.org/#dfn-set-entries) are the string names of the WGSL [language extensions](https://gpuweb.github.io/gpuweb/wgsl/#language-extension)
supported by the implementation (regardless of the adapter or device).

\[Exposed Window ,Worker ),SecureContext interface WGSLLanguageFeatures readonly setlike DOMString 

##### 3.6.2.4. `GPUAdapterInfo`

[GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)

None of the members in [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)`""`. It is at the user
agent’s discretion which values to reveal, and it is likely that on some devices none of the values
will be populated. As such, applications **must** be able to handle any possible [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)

The [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)[GPUAdapter.info](https://gpuweb.github.io#dom-gpuadapter-info)[GPUDevice.adapterInfo](https://gpuweb.github.io#dom-gpudevice-adapterinfo)[GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)

Note:
Though the [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)*once accessed*, an implementation may delay the decision on
what to expose for each attribute until the first time it is accessed.

Note:
Other [GPUAdapter](https://gpuweb.github.io#gpuadapter)[GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)**should** expose the same values unless a specific
event has increased the amount of identifying information the page is allowed to access.
(No such events are defined by this specification.)

 
[For privacy considerations, see](https://infra.spec.whatwg.org/#tracking-vector) [§ 2.2.6 Adapter Identifiers](https://gpuweb.github.io#privacy-adapter-identifiers).

\[Exposed Window ,Worker ),SecureContext interface GPUAdapterInfo readonly attribute DOMString vendor readonly attribute DOMString architecture readonly attribute DOMString device readonly attribute DOMString description readonly attribute unsigned long subgroupMinSize readonly attribute unsigned long subgroupMaxSize readonly attribute boolean isFallbackAdapter 

[GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)

- `vendor`,  of type [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString), readonly
- The name of the vendor of the [adapter](https://gpuweb.github.io#adapter), if available. Empty string otherwise.
- `architecture`,  of type [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString), readonly
- The name of the family or class of GPUs the [adapter](https://gpuweb.github.io#adapter) belongs to, if available. Empty
string otherwise.
- `device`,  of type [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString), readonly
- A vendor-specific identifier for the [adapter](https://gpuweb.github.io#adapter), if available. Empty string otherwise.
Note: This is a value that represents the type of adapter. For example, it may be a
[PCI device ID](https://pcisig.com/). It does not uniquely identify a given piece of
hardware like a serial number.
- `description`,  of type [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString), readonly
- A human readable string describing the [adapter](https://gpuweb.github.io#adapter) as reported by the driver, if available.
Empty string otherwise.
Note: Because no formatting is applied to [description](https://gpuweb.github.io#dom-gpuadapterinfo-description)[GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)
- `subgroupMinSize`,  of type [unsigned long](https://webidl.spec.whatwg.org/#idl-unsigned-long), readonly
- If the ["subgroups"](https://gpuweb.github.io#subgroups)[subgroup size](https://gpuweb.github.io/gpuweb/wgsl/#subgroup-size) for the [adapter](https://gpuweb.github.io#adapter).
- `subgroupMaxSize`,  of type [unsigned long](https://webidl.spec.whatwg.org/#idl-unsigned-long), readonly
- If the ["subgroups"](https://gpuweb.github.io#subgroups)[subgroup size](https://gpuweb.github.io/gpuweb/wgsl/#subgroup-size) for the [adapter](https://gpuweb.github.io#adapter).
- `isFallbackAdapter`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), readonly
- Whether the [adapter](https://gpuweb.github.io#adapter) is a [fallback adapter](https://gpuweb.github.io#fallback-adapter).

[adapter](https://gpuweb.github.io#adapter)

`adapter`, run the following

[content timeline](https://gpuweb.github.io#content-timeline)steps:

1. Let `adapterInfo` be a new [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)
2. If the vendor is known, set `adapterInfo`.[vendor](https://gpuweb.github.io#dom-gpuadapterinfo-vendor)`adapter`’s vendor as a [normalized identifier string](https://gpuweb.github.io#normalized-identifier-string). To preserve privacy, the user
agent may instead set `adapterInfo`.[vendor](https://gpuweb.github.io#dom-gpuadapterinfo-vendor)[normalized identifier string](https://gpuweb.github.io#normalized-identifier-string).
3. If |the architecture is known, set `adapterInfo`.[architecture](https://gpuweb.github.io#dom-gpuadapterinfo-architecture)[normalized identifier string](https://gpuweb.github.io#normalized-identifier-string) representing the family or class of adapters to which `adapter` belongs. To preserve privacy, the user agent may instead set `adapterInfo`.[architecture](https://gpuweb.github.io#dom-gpuadapterinfo-architecture)[normalized identifier string](https://gpuweb.github.io#normalized-identifier-string).
4. If the device is known, set `adapterInfo`.[device](https://gpuweb.github.io#dom-gpuadapterinfo-device)[normalized identifier string](https://gpuweb.github.io#normalized-identifier-string) representing a vendor-specific identifier for `adapter`.
To preserve privacy, the user agent may instead set `adapterInfo`.[device](https://gpuweb.github.io#dom-gpuadapterinfo-device)[normalized identifier string](https://gpuweb.github.io#normalized-identifier-string).
5. If a description is known, set `adapterInfo`.[description](https://gpuweb.github.io#dom-gpuadapterinfo-description)`adapter` as reported by the driver. To preserve privacy, the user agent may
instead set `adapterInfo`.[description](https://gpuweb.github.io#dom-gpuadapterinfo-description)
6. If ["subgroups"](https://gpuweb.github.io#subgroups)[subgroupMinSize](https://gpuweb.github.io#dom-gpuadapterinfo-subgroupminsize)
Note: To preserve privacy, the user agent may choose to not support some features or provide values for the property which do not distinguish different devices, but are still usable (e.g. use the default value of 4 for all devices).
7. If ["subgroups"](https://gpuweb.github.io#subgroups)[subgroupMaxSize](https://gpuweb.github.io#dom-gpuadapterinfo-subgroupmaxsize)
Note: To preserve privacy, the user agent may choose to not support some features or provide values for the property which do not distinguish different devices, but are still usable (e.g. use the default value of 128 for all devices).
8. Set `adapterInfo`.[isFallbackAdapter](https://gpuweb.github.io#dom-gpuadapterinfo-isfallbackadapter)`adapter`.[\[\[fallback\]\]](https://gpuweb.github.io#dom-adapter-fallback-slot)
9. Return `adapterInfo`.

`[a-z0-9]+(-[a-z0-9]+)*`

### 3.7. Feature Detection

*This section is non-normative.*

Fully implementing this specification requires implementation of everything it specifies, except
where otherwise stated (like [§ 3.6 Optional Capabilities](https://gpuweb.github.io#optional-capabilities)).

However, since new "core" additions are added to this specification before being exposed by implementations, many features are designed to be feature-detectable by applications:

- Interface support can be detected with `typeof InterfaceName !== 'undefined'`.
- Method and attribute support can be detected with `'itemName' in InterfaceName.prototype`.
- New dictionary members, if they need to be detectable, generally document a specific mechanism for feature detection. For example: 
  - [unclippedDepth](https://gpuweb.github.io#dom-gpuprimitivestate-unclippeddepth)["depth-clip-control"](https://gpuweb.github.io#depth-clip-control)
  - Canvas support for [toneMapping](https://gpuweb.github.io#dom-gpucanvasconfiguration-tonemapping)[getConfiguration()](https://gpuweb.github.io#dom-gpucanvascontext-getconfiguration)

### 3.8. Extension Documents

"Extension Documents" are additional documents which describe new functionality which is
non-normative and **not part of the WebGPU/WGSL specifications**.
They describe functionality that builds upon these specifications, often including one or more new
API [feature](https://gpuweb.github.io#feature) flags and/or WGSL `enable` directives, or interactions with other draft
web specifications.

WebGPU implementations **must not** expose extension functionality; doing so is a spec violation.
New functionality does not become part of the WebGPU standard until it is integrated
into the WebGPU specification (this document) and/or WGSL specification.

### 3.9. Origin Restrictions

WebGPU allows accessing image data stored in images, videos, and canvases. Restrictions are imposed on the use of cross-domain media, because shaders can be used to indirectly deduce the contents of textures which have been uploaded to the GPU.

WebGPU disallows uploading an image source if it [is not origin-clean](https://html.spec.whatwg.org/multipage/canvas.html#the-image-argument-is-not-origin-clean).

This also implies that the [origin-clean](https://html.spec.whatwg.org/multipage/canvas.html#concept-canvas-origin-clean) flag for a
canvas rendered using WebGPU will never be set to `false`.

For more information on issuing CORS requests for image and video elements, consult:

### 3.10. Task Sources

#### 3.10.1. WebGPU Task Source

WebGPU defines a new [task source](https://html.spec.whatwg.org/multipage/webappapis.html#task-source) called the WebGPU task source.
It is used for the [uncapturederror](https://gpuweb.github.io#eventdef-gpudevice-uncapturederror)[GPUDevice](https://gpuweb.github.io#gpudevice)[lost](https://gpuweb.github.io#dom-gpudevice-lost)

[GPUDevice](https://gpuweb.github.io#gpudevice)
`device`, with a series of steps

`steps`on the

[content timeline](https://gpuweb.github.io#content-timeline):

1. [Queue a global task](https://html.spec.whatwg.org/multipage/webappapis.html#queue-a-global-task) on the [WebGPU task source](https://gpuweb.github.io#webgpu-task-source), with the global object that was used
to create `device`, and the steps `steps`.

#### 3.10.2. Automatic Expiry Task Source

WebGPU defines a new [task source](https://html.spec.whatwg.org/multipage/webappapis.html#task-source) called the [automatic expiry task source](https://gpuweb.github.io#automatic-expiry-task-source).
It is used for the automatic, timed expiry (destruction) of certain objects:

- [GPUTexture](https://gpuweb.github.io#gputexture)[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
- [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[HTMLVideoElement](https://html.spec.whatwg.org/multipage/media.html#htmlvideoelement)

[GPUDevice](https://gpuweb.github.io#gpudevice)
`device`and a series of steps

`steps`on the

[content timeline](https://gpuweb.github.io#content-timeline):

1. [Queue a global task](https://html.spec.whatwg.org/multipage/webappapis.html#queue-a-global-task) on the [automatic expiry task source](https://gpuweb.github.io#automatic-expiry-task-source), with the global object that
was used to create `device`, and the steps `steps`.

Tasks from the [automatic expiry task source](https://gpuweb.github.io#automatic-expiry-task-source) **should** be processed with high priority; in
particular, once queued, they **should** run before user-defined (JavaScript) tasks.

Implementation note:
    It is valid to implement a high-priority expiry "task" by instead inserting additional steps at
    a fixed point inside the [event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) rather than running an actual task.

### 3.11. Color Spaces and Encoding

WebGPU does not provide color management. All values within WebGPU (such as texture elements) are raw numeric values, not color-managed color values.

WebGPU *does* interface with color-managed outputs (via [GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)[copyExternalImageToTexture()](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture)[importExternalTexture()](https://gpuweb.github.io#dom-gpudevice-importexternaltexture)

WebGPU allows all of the color spaces in the [PredefinedColorSpace](https://html.spec.whatwg.org/multipage/canvas.html#predefinedcolorspace)

[GPUTexture](https://gpuweb.github.io#gputexture)`-srgb` formats,
    which despite their are not *tagged*with an sRGB color space (like those described by

[PredefinedColorSpace](https://html.spec.whatwg.org/multipage/canvas.html#predefinedcolorspace)
[srgb](https://drafts.csswg.org/css-color-4/#valdef-color-srgb)and

[srgb-linear](https://drafts.csswg.org/css-color-4/#valdef-color-srgb-linear)).

However, `-srgb` texture formats *do* have gamma-encoding/decoding properties which are
    algorithmically close to those used for gamma encoding in ["srgb"](https://html.spec.whatwg.org/multipage/canvas.html#dom-predefinedcolorspace-srgb)["display-p3"](https://html.spec.whatwg.org/multipage/canvas.html#dom-predefinedcolorspace-display-p3)`-srgb`
    format texture, which will gamma-encode the value when it is written.
    Then, the value in the texture will be correctly encoded for use on a
    ["srgb"](https://html.spec.whatwg.org/multipage/canvas.html#dom-predefinedcolorspace-srgb)

It is similarly possible to take advantage of these properties using
    [copyExternalImageToTexture()](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture)

An out-of-gamut premultiplied RGBA value is one where any of the R/G/B channel values
exceeds the alpha channel value. For example, the premultiplied sRGB RGBA value \[1.0, 0, 0, 0.5\]
represents the (unpremultiplied) color \[2, 0, 0\] with 50% alpha, written `rgb(srgb 2 0 0 / 50%)` in CSS.
Just like any color value outside the sRGB color gamut, this is a well defined point in the extended color space
(except when alpha is 0, in which case there is no color).
However, when such values are output to a visible canvas, the result is undefined
(see [GPUCanvasAlphaMode](https://gpuweb.github.io#gpucanvasalphamode)["premultiplied"](https://gpuweb.github.io#dom-gpucanvasalphamode-premultiplied)

#### 3.11.1. Color Space Conversions

A color is converted between spaces by translating its representation in one space to a representation in another according to the definitions above.

If the source value has fewer than 4 RGBA channels, the missing green/blue/alpha channels are set to
`0, 0, 1`, respectively, before converting for color space/encoding and alpha premultiplication.
After conversion, if the destination needs fewer than 4 channels, the additional channels
are ignored.

Note:
Grayscale images generally represent RGB values `(V, V, V)`, or RGBA values `(V, V, V, A)` in their color space.

Colors are not lossily clamped during conversion: converting from one color space to another
will result in values outside the range \[0, 1\] if the source color values were outside the range
of the destination color space’s gamut. For an sRGB destination, for example, this can occur if the
source is rgba16float, in a wider color space like Display-P3, or is premultiplied and contains
[out-of-gamut values](https://gpuweb.github.io#out-of-gamut-premultiplied-rgba-value).

Similarly, if the source value has a high bit depth (e.g. PNG with 16 bits per component) or
extended range (e.g. canvas with `float16` storage), these colors are preserved through color space
conversion, with intermediate computations having at least the precision of the source.

#### 3.11.2. Color Space Conversion Elision

If the source and destination of a color space/encoding conversion are the same, then conversion
is not necessary. In general, if any given step of the conversion is an identity function (no-op),
implementations **should** elide it, for performance.

For optimal performance, applications **should** set their color space and encoding
options so that the number of necessary conversions is minimized throughout the process.
For various image sources of [GPUCopyExternalImageSourceInfo](https://gpuweb.github.io#gpucopyexternalimagesourceinfo)

- [ImageBitmap](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#imagebitmap)
  - Premultiplication is controlled via [premultiplyAlpha](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#dom-imagebitmapoptions-premultiplyalpha)
  - Color space is controlled via [colorSpaceConversion](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#dom-imagebitmapoptions-colorspaceconversion)
- 2d canvas: 
  - [Always premultiplied](https://html.spec.whatwg.org/multipage/canvas.html#premultiplied-alpha-and-the-2d-rendering-context).
  - Color space is controlled via the [colorSpace](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvasrenderingcontext2dsettings-colorspace)
- WebGL canvas: 
  - Premultiplication is controlled via the `premultipliedAlpha` option in [WebGLContextAttributes](https://www.khronos.org/registry/webgl/specs/latest/1.0/#WEBGLCONTEXTATTRIBUTES)
  - Color space is controlled via the [WebGLRenderingContextBase](https://www.khronos.org/registry/webgl/specs/latest/1.0/#WEBGLRENDERINGCONTEXTBASE)[drawingBufferColorSpace](https://www.khronos.org/registry/webgl/specs/latest/1.0/#DOM-WebGLRenderingContext-drawingBufferColorSpace)

Note: Check browser implementation support for these features before relying on them.

### 3.12. Numeric conversions from JavaScript to WGSL

Several parts of the WebGPU API ([pipeline-overridable](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable) [constants](https://gpuweb.github.io#dom-gpuprogrammablestage-constants)[double](https://webidl.spec.whatwg.org/#idl-double)[float](https://webidl.spec.whatwg.org/#idl-float)`bool`, `i32`, `u32`, `f32`, `f16`).

`idlValue`of type

[double](https://webidl.spec.whatwg.org/#idl-double)[float](https://webidl.spec.whatwg.org/#idl-float)
`T`, possibly throwing a

[TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)
[device timeline](https://gpuweb.github.io#device-timeline)steps:

Note: This [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)[device timeline](https://gpuweb.github.io#device-timeline) and never surfaced to JavaScript.

1. [Assert](https://infra.spec.whatwg.org/#assert) `idlValue` is a finite value, since it is not [unrestricted double](https://webidl.spec.whatwg.org/#idl-unrestricted-double)[unrestricted float](https://webidl.spec.whatwg.org/#idl-unrestricted-float)
2. Let `v` be the ECMAScript Number resulting from [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) converting `idlValue` to [an ECMAScript value](https://webidl.spec.whatwg.org/#dfn-convert-idl-to-javascript-value).
  - If `T` is `bool`
  - Return the WGSL `bool` value corresponding to the result of [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) converting `v` to [an IDL value](https://webidl.spec.whatwg.org/#dfn-convert-ecmascript-to-idl-value) of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
Note:
This algorithm is called after the conversion from an ECMAScript value to an IDL
[double](https://webidl.spec.whatwg.org/#idl-double)[float](https://webidl.spec.whatwg.org/#idl-float)`[]` or `{}`, then the WGSL `bool` result may be different
than if the ECMAScript value had been converted to IDL [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
  - If `T` is `i32`
  - Return the WGSL `i32` value corresponding to the result of [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) converting `v` to [an IDL value](https://webidl.spec.whatwg.org/#dfn-convert-ecmascript-to-idl-value) of type \[[EnforceRange](https://webidl.spec.whatwg.org/#EnforceRange)[long](https://webidl.spec.whatwg.org/#idl-long)
  - If `T` is `u32`
  - Return the WGSL `u32` value corresponding to the result of [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) converting `v` to [an IDL value](https://webidl.spec.whatwg.org/#dfn-convert-ecmascript-to-idl-value) of type \[[EnforceRange](https://webidl.spec.whatwg.org/#EnforceRange)[unsigned long](https://webidl.spec.whatwg.org/#idl-unsigned-long)
  - If `T` is `f32`
  - Return the WGSL `f32` value corresponding to the result of [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) converting `v` to [an IDL value](https://webidl.spec.whatwg.org/#dfn-convert-ecmascript-to-idl-value) of type [float](https://webidl.spec.whatwg.org/#idl-float)
  - If `T` is `f16`
    1. Let `wgslF32` be the WGSL `f32` value corresponding to the result of [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) converting `v` to [an IDL value](https://webidl.spec.whatwg.org/#dfn-convert-ecmascript-to-idl-value) of type [float](https://webidl.spec.whatwg.org/#idl-float)
    2. Return `f16(`, the result of `wgslF32`)[!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) converting the WGSL `f32` value
to `f16` as defined in [WGSL floating point conversion](https://gpuweb.github.io/gpuweb/wgsl/#floating-point-conversion).
Note: As long as the value is in-range of `f32`, no error is thrown, even if the
value is out-of-range of `f16`.

[GPUColor](https://gpuweb.github.io#typedefdef-gpucolor)
`color`to a texel value of texture format

`format`, possibly throwing a

[TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)
[device timeline](https://gpuweb.github.io#device-timeline)steps:

Note: This [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)[device timeline](https://gpuweb.github.io#device-timeline) and never surfaced to JavaScript.

1. If the components of `format` ([assert](https://infra.spec.whatwg.org/#assert) they all have the same type) are:
  - floating-point types or normalized types
  - Let `T` be `f32`.
  - signed integer types
  - Let `T` be `i32`.
  - unsigned integer types
  - Let `T` be `u32`.
2. Let `wgslColor` be a WGSL value of type `vec4<`, where the 4
components are the RGBA channels of `T`>`color`, each [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) converted [to WGSL type](https://gpuweb.github.io#abstract-opdef-to-wgsl-type) `T`.
3. Convert `wgslColor` to `format` using the same conversion rules as the [§ 23.2.7 Output Merging](https://gpuweb.github.io#output-merging) step, and return the result.
Note:
For non-integer types, the exact choice of value is [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined).
For normalized types, the value is clamped to the range of the type.

Note:
    In other words, the value written will be as if it was written by a WGSL shader that
    outputs the value represented as a `vec4` of `f32`, `i32`, or `u32`.

## 4. Initialization

### 4.1. navigator.gpu

A [GPU](https://gpuweb.github.io#gpu)[Window](https://html.spec.whatwg.org/multipage/nav-history-apis.html#window)[WorkerGlobalScope](https://html.spec.whatwg.org/multipage/workers.html#workerglobalscope)[Navigator](https://html.spec.whatwg.org/multipage/system-state.html#navigator)[WorkerNavigator](https://html.spec.whatwg.org/multipage/workers.html#workernavigator)`navigator.gpu`:

interface mixin NavigatorGPU SameObject SecureContext readonly attribute GPU gpu Navigator includes NavigatorGPU WorkerNavigator includes NavigatorGPU 

[NavigatorGPU](https://gpuweb.github.io#navigatorgpu)

- `gpu`,  of type [GPU](https://gpuweb.github.io#gpu), readonly
- A global singleton providing top-level entry points like [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)

### 4.2. GPU

`GPU` is the entry point to WebGPU.

\[Exposed Window ,Worker ),SecureContext interface GPU Promise GPUAdapter requestAdapter optional GPURequestAdapterOptions options GPUTextureFormat getPreferredCanvasFormat SameObject readonly attribute WGSLLanguageFeatures wgslLanguageFeatures 

[GPU](https://gpuweb.github.io#gpu)

- `requestAdapter(options)`
- Requests an [adapter](https://gpuweb.github.io#adapter) from the user agent.
The user agent chooses whether to return an adapter, and, if so,
chooses according to the provided options.
**Called on:** [GPU](https://gpuweb.github.io#gpu)
`this`. **Arguments:**
Arguments for the [GPU.requestAdapter(options)](https://gpuweb.github.io#dom-gpu-requestadapter) method.Parameter Type Nullable Optional Description `options`
[GPURequestAdapterOptions](https://gpuweb.github.io#dictdef-gpurequestadapteroptions)
✘ ✔ Criteria used to select the adapter. **Returns:** [Promise](https://webidl.spec.whatwg.org/#idl-promise)[GPUAdapter](https://gpuweb.github.io#gpuadapter)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `contentTimeline` be the current [Content timeline](https://gpuweb.github.io#content-timeline).
  2. Let `promise` be [a new promise](https://webidl.spec.whatwg.org/#a-new-promise).
  3. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  4. Return `promise`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. All of the requirements in the following steps `must` be met.
    1. `options`.[featureLevel](https://gpuweb.github.io#dom-gpurequestadapteroptions-featurelevel)`must` be
a [feature level string](https://gpuweb.github.io#feature-level-string).
If any are unmet: 
    1. Let `adapter` be `null`, issue the `resolution steps` on `contentTimeline`, and return.
  2. If `options`.[featureLevel](https://gpuweb.github.io#dom-gpurequestadapteroptions-featurelevel)["compatibility"](https://gpuweb.github.io#feature-level-string-compatibility):
    1. Set `options`.[featureLevel](https://gpuweb.github.io#dom-gpurequestadapteroptions-featurelevel)["compatibility"](https://gpuweb.github.io#feature-level-string-compatibility) if the user agent chooses
to support it, or ["core"](https://gpuweb.github.io#feature-level-string-core) if not.
Note: This doesn’t modify the JavaScript object passed by the application.
  3. Set `adapter` to either:
    - A new [adapter](https://gpuweb.github.io#adapter) object chosen according to
the rules in [§ 4.2.2 Adapter Selection](https://gpuweb.github.io#adapter-selection) and the criteria in `options`,
adhering to [§ 4.2.1 Adapter Capability Guarantees](https://gpuweb.github.io#adapter-capability-guarantees), with the capabilities
determined in an [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined) way by the user agent.
    - `null`, if the user agent is unable to return an adapter, or makes an [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined) choice not to return an adapter.
If an `adapter` is returned, initialize its properties according to their
definitions.
    1. Set `adapter`.[\[\[limits\]\]](https://gpuweb.github.io#dom-adapter-limits-slot)`adapter`.[\[\[features\]\]](https://gpuweb.github.io#dom-adapter-features-slot)
    2. If `adapter` meets the criteria of a [fallback adapter](https://gpuweb.github.io#fallback-adapter) set `adapter`.[\[\[fallback\]\]](https://gpuweb.github.io#dom-adapter-fallback-slot)`true`. Otherwise, set it to `false`.
    3. Set `adapter`.[\[\[xrCompatible\]\]](https://gpuweb.github.io#dom-adapter-xrcompatible-slot)`options`.[xrCompatible](https://gpuweb.github.io#dom-gpurequestadapteroptions-xrcompatible)
    4. Set `adapter`.[\[\[default feature level\]\]](https://gpuweb.github.io#dom-adapter-default-feature-level-slot)`options`.[featureLevel](https://gpuweb.github.io#dom-gpurequestadapteroptions-featurelevel)
  4. Issue the `resolution steps` on `contentTimeline`.
[Content timeline](https://gpuweb.github.io#content-timeline) `resolution steps`:
  1. If `adapter` is not `null`:
    1. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise` with a new [GPUAdapter](https://gpuweb.github.io#gpuadapter)`adapter`.
Otherwise: 
    1. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise` with `null`.
- `getPreferredCanvasFormat()`
- Returns an optimal [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)["rgba8unorm"](https://gpuweb.github.io#dom-gputextureformat-rgba8unorm)["bgra8unorm"](https://gpuweb.github.io#dom-gputextureformat-bgra8unorm)
The returned value can be passed as the [format](https://gpuweb.github.io#dom-gpucanvasconfiguration-format)[configure()](https://gpuweb.github.io#dom-gpucanvascontext-configure)[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
Note: Canvases which are not displayed to the screen may or may not benefit from using this format. **Called on:** [GPU](https://gpuweb.github.io#gpu)
**Returns:** [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Return either ["rgba8unorm"](https://gpuweb.github.io#dom-gputextureformat-rgba8unorm)["bgra8unorm"](https://gpuweb.github.io#dom-gputextureformat-bgra8unorm)

[GPU](https://gpuweb.github.io#gpu)

- `wgslLanguageFeatures`,  of type [WGSLLanguageFeatures](https://gpuweb.github.io#gpuwgsllanguagefeatures), readonly
- The names of supported WGSL [language extensions](https://gpuweb.github.io/gpuweb/wgsl/#language-extension).
Supported language extensions are automatically enabled.

[Adapters](https://gpuweb.github.io#adapter) **may** [expire](https://gpuweb.github.io#abstract-opdef-expire) at any time. Upon any change in the system’s state that could affect
the result of any [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)**should** [expire](https://gpuweb.github.io#abstract-opdef-expire) all
previously-returned [adapters](https://gpuweb.github.io#adapter). For example:

- A physical adapter is added/removed (via plug/unplug, driver update, hang recovery, etc.)
- The system’s power configuration has changed (laptop unplugged, power settings changed, etc.)

Note:
User agents may choose to [expire](https://gpuweb.github.io#abstract-opdef-expire) [adapters](https://gpuweb.github.io#adapter) often, even when there has been no system
state change (e.g. seconds or minutes after the adapter was created).
This can help obfuscate real system state changes, and make developers more aware that calling
[requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)[requestDevice()](https://gpuweb.github.io#dom-gpuadapter-requestdevice)

#### 4.2.1. Adapter Capability Guarantees

Any [GPUAdapter](https://gpuweb.github.io#gpuadapter)[requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)

- At least one of the following must be true: 
  - ["texture-compression-bc"](https://gpuweb.github.io#texture-compression-bc)
  - Both ["texture-compression-etc2"](https://gpuweb.github.io#texture-compression-etc2)["texture-compression-astc"](https://gpuweb.github.io#texture-compression-astc)
- For each `feature` supported by the device:
  - The adapter must also support all of the features required (directly or indirectly)
by `feature`, as defined in [§ 3.6.1.2 Feature Dependencies](https://gpuweb.github.io#feature-dependencies).
- All supported limits must be either the [default](https://gpuweb.github.io#limit-default) value or [better](https://gpuweb.github.io#limit-better).
- All [alignment-class](https://gpuweb.github.io#limit-class-alignment) limits must be powers of 2.
- [maxBindingsPerBindGroup](https://gpuweb.github.io#dom-supported-limits-maxbindingsperbindgroup)[max bindings per shader stage](https://gpuweb.github.io#max-bindings-per-shader-stage) × [max shader stages per pipeline](https://gpuweb.github.io#max-shader-stages-per-pipeline)), where:
  - max bindings per shader stage is
([maxSampledTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxsampledtexturespershaderstage)[maxSamplersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxsamplerspershaderstage)[maxStorageBuffersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebufferspershaderstage)[maxStorageTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturespershaderstage)[maxUniformBuffersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxuniformbufferspershaderstage)
  - max shader stages per pipeline is `2`, because a [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
Note: [maxBindingsPerBindGroup](https://gpuweb.github.io#dom-supported-limits-maxbindingsperbindgroup)
- [maxBindGroups](https://gpuweb.github.io#dom-supported-limits-maxbindgroups)[maxBindGroupsPlusVertexBuffers](https://gpuweb.github.io#dom-supported-limits-maxbindgroupsplusvertexbuffers)
- [maxVertexBuffers](https://gpuweb.github.io#dom-supported-limits-maxvertexbuffers)[maxBindGroupsPlusVertexBuffers](https://gpuweb.github.io#dom-supported-limits-maxbindgroupsplusvertexbuffers)
- [minUniformBufferOffsetAlignment](https://gpuweb.github.io#dom-supported-limits-minuniformbufferoffsetalignment)[minStorageBufferOffsetAlignment](https://gpuweb.github.io#dom-supported-limits-minstoragebufferoffsetalignment)
Note: 32 bytes would be the alignment of `vec4<f64>`. See [WebGPU Shading Language § 14.4.1 Alignment and Size](https://gpuweb.github.io/gpuweb/wgsl/#alignment-and-size).
- [maxUniformBufferBindingSize](https://gpuweb.github.io#dom-supported-limits-maxuniformbufferbindingsize)[maxBufferSize](https://gpuweb.github.io#dom-supported-limits-maxbuffersize)
- [maxStorageBufferBindingSize](https://gpuweb.github.io#dom-supported-limits-maxstoragebufferbindingsize)[maxBufferSize](https://gpuweb.github.io#dom-supported-limits-maxbuffersize)
- [maxStorageBufferBindingSize](https://gpuweb.github.io#dom-supported-limits-maxstoragebufferbindingsize)
- [maxVertexBufferArrayStride](https://gpuweb.github.io#dom-supported-limits-maxvertexbufferarraystride)
- [maxComputeWorkgroupSizeX](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizex)[maxComputeInvocationsPerWorkgroup](https://gpuweb.github.io#dom-supported-limits-maxcomputeinvocationsperworkgroup)
- [maxComputeWorkgroupSizeY](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizey)[maxComputeInvocationsPerWorkgroup](https://gpuweb.github.io#dom-supported-limits-maxcomputeinvocationsperworkgroup)
- [maxComputeWorkgroupSizeZ](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizez)[maxComputeInvocationsPerWorkgroup](https://gpuweb.github.io#dom-supported-limits-maxcomputeinvocationsperworkgroup)
- [maxComputeInvocationsPerWorkgroup](https://gpuweb.github.io#dom-supported-limits-maxcomputeinvocationsperworkgroup)[maxComputeWorkgroupSizeX](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizex)[maxComputeWorkgroupSizeY](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizey)[maxComputeWorkgroupSizeZ](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizez)
- [maxImmediateSize](https://gpuweb.github.io#dom-supported-limits-maximmediatesize)

#### 4.2.2. Adapter Selection

`GPURequestAdapterOptions`
provides hints to the user agent indicating what
configuration is suitable for the application.

dictionary GPURequestAdapterOptions DOMString featureLevel GPUPowerPreference powerPreference boolean forceFallbackAdapter false ;boolean xrCompatible false ; };

enum GPUPowerPreference "low-power" "high-performance" 

[GPURequestAdapterOptions](https://gpuweb.github.io#dictdef-gpurequestadapteroptions)

- `featureLevel`,  of type [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString), defaulting to `"core"`
- Requests an adapter that supports at least a particular set of [capabilities](https://gpuweb.github.io#capabilities).
This influences the [\[\[default feature level\]\]](https://gpuweb.github.io#dom-adapter-default-feature-level-slot)[requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)[a new device](https://gpuweb.github.io#a-new-device)".
If the implementation or system does not support all of the capabilities in the
requested feature level, [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)`null`.
Note:
Applications should typically make a single [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)[requestDevice()](https://gpuweb.github.io#dom-gpuadapter-requestdevice)
The allowed feature level string values are: 
  - "core"
  - The following set of capabilities: 
    - The [Default](https://gpuweb.github.io#limit-default) limits.
    - ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
Note:
Adapters with this [\[\[default feature level\]\]](https://gpuweb.github.io#dom-adapter-default-feature-level-slot)
  - "compatibility"
  - The following set of capabilities: 
    - The [Compatibility Mode Default](https://gpuweb.github.io#limit-compatibility-mode-default) limits.
    - No features. (It excludes the ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
If the implementation cannot enforce the stricter "Compatibility Mode"
validation rules, [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)["core"](https://gpuweb.github.io#feature-level-string-core).
Note:
Adapters with this [\[\[default feature level\]\]](https://gpuweb.github.io#dom-adapter-default-feature-level-slot)
- `powerPreference`,  of type [GPUPowerPreference](https://gpuweb.github.io#enumdef-gpupowerpreference)
- Optionally provides a hint indicating what class of [adapter](https://gpuweb.github.io#adapter) should be selected from
the system’s available adapters.
The value of this hint may influence which adapter is chosen, but it must not influence whether an adapter is returned or not. Note: The primary utility of this hint is to influence which GPU is used in a multi-GPU system. For instance, some laptops have a low-power integrated GPU and a high-performance discrete GPU. This hint may also affect the power configuration of the selected GPU to match the requested power preference. Note:
Depending on the exact hardware configuration, such as battery status and attached displays
or removable GPUs, the user agent may select different [adapters](https://gpuweb.github.io#adapter) given the same power
preference.
Typically, given the same hardware configuration and state and `powerPreference`, the user agent is likely to select the same adapter.
It must be one of the following values: 
  - `undefined` (or not present)
  - Provides no hint to the user agent.
  - `"low-power"`
  - Indicates a request to prioritize power savings over performance. Note: Generally, content should use this if it is unlikely to be constrained by drawing performance; for example, if it renders only one frame per second, draws only relatively simple geometry with simple shaders, or uses a small HTML canvas element. Developers are encouraged to use this value if their content allows, since it may significantly improve battery life on portable devices.
  - `"high-performance"`
  - Indicates a request to prioritize performance over power consumption. Note:
By choosing this value, developers should be aware that, for [devices](https://gpuweb.github.io#device) created on the
resulting adapter, user agents are more likely to force device loss, in order to save
power by switching to a lower-power adapter.
Developers are encouraged to only specify this value if they believe it is absolutely
necessary, since it may significantly decrease battery life on portable devices.
- `forceFallbackAdapter`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- When set to `true` indicates that only a [fallback adapter](https://gpuweb.github.io#fallback-adapter) may be returned. If the user
agent does not support a [fallback adapter](https://gpuweb.github.io#fallback-adapter), will cause [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)`null`.
Note:
[requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)[fallback adapter](https://gpuweb.github.io#fallback-adapter) if [forceFallbackAdapter](https://gpuweb.github.io#dom-gpurequestadapteroptions-forcefallbackadapter)`false` and either no
other appropriate [adapter](https://gpuweb.github.io#adapter) is available or the user agent chooses to return a [fallback adapter](https://gpuweb.github.io#fallback-adapter). Developers that wish to prevent their applications from running on [fallback adapters](https://gpuweb.github.io#fallback-adapter) should check the [info](https://gpuweb.github.io#dom-gpuadapter-info)[isFallbackAdapter](https://gpuweb.github.io#dom-gpuadapterinfo-isfallbackadapter)[GPUDevice](https://gpuweb.github.io#gpudevice)
- `xrCompatible`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- When set to `true` indicates that the best [adapter](https://gpuweb.github.io#adapter) for rendering to a [WebXR session](https://www.w3.org/TR/webxr/#session) must be returned. If the user agent or system does not support [WebXR sessions](https://www.w3.org/TR/webxr/#session) then
adapter selection may ignore this value.
Note:
If [xrCompatible](https://gpuweb.github.io#dom-gpurequestadapteroptions-xrcompatible)`true` when the adapter is
requested, [GPUDevice](https://gpuweb.github.io#gpudevice)[WebXR sessions](https://www.w3.org/TR/webxr/#session).

[Requesting a](https://gpuweb.github.io#example-6808af9c)

["high-performance"](https://gpuweb.github.io#dom-gpupowerpreference-high-performance)[GPUAdapter](https://gpuweb.github.io#gpuadapter)
const gpuAdapter= await navigator. gpu. requestAdapter({ powerPreference: 'high-performance' }); 

### 4.3. `GPUAdapter`

A [GPUAdapter](https://gpuweb.github.io#gpuadapter)[adapter](https://gpuweb.github.io#adapter),
and describes its capabilities ([features](https://gpuweb.github.io#feature) and [limits](https://gpuweb.github.io#limit)).

To get a [GPUAdapter](https://gpuweb.github.io#gpuadapter)[requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)

\[Exposed Window ,Worker ),SecureContext interface GPUAdapter SameObject readonly attribute GPUSupportedFeatures features SameObject readonly attribute GPUSupportedLimits limits SameObject readonly attribute GPUAdapterInfo info Promise GPUDevice requestDevice optional GPUDeviceDescriptor descriptor 

[GPUAdapter](https://gpuweb.github.io#gpuadapter)[immutable properties](https://gpuweb.github.io#immutable-property)

- `features`,  of type [GPUSupportedFeatures](https://gpuweb.github.io#gpusupportedfeatures), readonly
- The set of values in `this`.[\[\[adapter\]\]](https://gpuweb.github.io#dom-gpuadapter-adapter-slot)[\[\[features\]\]](https://gpuweb.github.io#dom-adapter-features-slot)
- `limits`,  of type [GPUSupportedLimits](https://gpuweb.github.io#gpusupportedlimits), readonly
- The limits in `this`.[\[\[adapter\]\]](https://gpuweb.github.io#dom-gpuadapter-adapter-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-adapter-limits-slot)
- `info`,  of type [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo), readonly
- Information about the physical adapter underlying this [GPUAdapter](https://gpuweb.github.io#gpuadapter)
For a given [GPUAdapter](https://gpuweb.github.io#gpuadapter)[GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)
The same object is returned each time. To create that object for the first time: **Called on:** [GPUAdapter](https://gpuweb.github.io#gpuadapter)
`this`. **Returns:** [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Return a [new adapter info](https://gpuweb.github.io#abstract-opdef-new-adapter-info) for `this`.[\[\[adapter\]\]](https://gpuweb.github.io#dom-gpuadapter-adapter-slot)
- `[[adapter]]`, of type [adapter](https://gpuweb.github.io#adapter), readonly
- The [adapter](https://gpuweb.github.io#adapter) to which this [GPUAdapter](https://gpuweb.github.io#gpuadapter)

[GPUAdapter](https://gpuweb.github.io#gpuadapter)

- `requestDevice(descriptor)`
- Requests a [device](https://gpuweb.github.io#device) from the [adapter](https://gpuweb.github.io#adapter).
This is a one-time action: if a device is returned successfully,
the adapter becomes ["consumed"](https://gpuweb.github.io#dom-adapter-state-consumed)
**Called on:** [GPUAdapter](https://gpuweb.github.io#gpuadapter)
`this`. **Arguments:**
Arguments for the [GPUAdapter.requestDevice(descriptor)](https://gpuweb.github.io#dom-gpuadapter-requestdevice) method.Parameter Type Nullable Optional Description `descriptor`
[GPUDeviceDescriptor](https://gpuweb.github.io#gpudevicedescriptor)
✘ ✔ Description of the [GPUDevice](https://gpuweb.github.io#gpudevice)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `contentTimeline` be the current [Content timeline](https://gpuweb.github.io#content-timeline).
  2. Let `promise` be [a new promise](https://webidl.spec.whatwg.org/#a-new-promise).
  3. Let `adapter` be `this`.[\[\[adapter\]\]](https://gpuweb.github.io#dom-gpuadapter-adapter-slot)
  4. Issue the `initialization steps` to the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  5. Return `promise`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. If any of the following requirements are unmet: 
    - The set of values in `descriptor`.[requiredFeatures](https://gpuweb.github.io#dom-gpudevicedescriptor-requiredfeatures)`adapter`.[\[\[features\]\]](https://gpuweb.github.io#dom-adapter-features-slot)
Then issue the following steps on `contentTimeline` and return:
[Content timeline](https://gpuweb.github.io#content-timeline) steps: Note: This is the same error that is produced if a feature name isn’t known
by the browser at all (in its [GPUFeatureName](https://gpuweb.github.io#gpufeaturename)
  2. All of the requirements in the following steps `must` be met.
    1. `adapter`.[\[\[state\]\]](https://gpuweb.github.io#dom-adapter-state-slot)["consumed"](https://gpuweb.github.io#dom-adapter-state-consumed)
    2. For each \[`key`, `value`\] in `descriptor`.[requiredLimits](https://gpuweb.github.io#dom-gpudevicedescriptor-requiredlimits)`value` is not `undefined`:
      1. `key must` be the name of a member of [supported limits](https://gpuweb.github.io#supported-limits).
      2. `value must` be no [better](https://gpuweb.github.io#limit-better) than `adapter`.[\[\[limits\]\]](https://gpuweb.github.io#dom-adapter-limits-slot)`key`\].
      3. If `key`’s [class](https://gpuweb.github.io#limit-class) is [alignment](https://gpuweb.github.io#limit-class-alignment), `value must` be a power of 2 less than 2<sup>32</sup>.
Note:
User agents should consider issuing developer-visible warnings when
`key` is not recognized, even when `value` is `undefined`.
If any are unmet, issue the following steps on `contentTimeline` and return:
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `promise` with an [OperationError](https://webidl.spec.whatwg.org/#operationerror)
  3. If `adapter`.[\[\[state\]\]](https://gpuweb.github.io#dom-adapter-state-slot)["expired"](https://gpuweb.github.io#dom-adapter-state-expired)
    1. Let `device` be a new [device](https://gpuweb.github.io#device).
    2. [Lose the device](https://gpuweb.github.io#lose-the-device)(`device`, ["unknown"](https://gpuweb.github.io#dom-gpudevicelostreason-unknown)
    3. [Assert](https://infra.spec.whatwg.org/#assert) `adapter`.[\[\[state\]\]](https://gpuweb.github.io#dom-adapter-state-slot)["expired"](https://gpuweb.github.io#dom-adapter-state-expired)
Note:
User agents should consider issuing developer-visible warnings in
most or all cases when this occurs. Applications should perform
reinitialization logic starting with [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)
Otherwise: 
    1. Let `device` be the result of creating [a new device](https://gpuweb.github.io#a-new-device) from `adapter` with `descriptor`.
    2. Set `adapter`.[\[\[adapter\]\]](https://gpuweb.github.io#dom-gpuadapter-adapter-slot)[\[\[state\]\]](https://gpuweb.github.io#dom-adapter-state-slot)
["consumed"](https://gpuweb.github.io#dom-adapter-state-consumed)
  1. Issue the subsequent steps on `contentTimeline`.
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `gpuDevice` be a new [GPUDevice](https://gpuweb.github.io#gpudevice)
  2. Set `gpuDevice`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)`device`.
  3. Set `device`.[\[\[content device\]\]](https://gpuweb.github.io#dom-device-content-device-slot)`gpuDevice`.
  4. Set `gpuDevice`.[label](https://gpuweb.github.io#dom-gpuobjectbase-label)`descriptor`.[label](https://gpuweb.github.io#dom-gpuobjectdescriptorbase-label)
  5. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise` with `gpuDevice`.
Note:
If the device is already lost because the adapter could not fulfill the request,
`device`.[lost](https://gpuweb.github.io#dom-gpudevice-lost)`promise` resolves.

[Requesting a](https://gpuweb.github.io#example-c8b5b8a0)

[GPUDevice](https://gpuweb.github.io#gpudevice)
const gpuAdapter= await navigator. gpu. requestAdapter(); const gpuDevice= await gpuAdapter. requestDevice(); 

#### 4.3.1. `GPUDeviceDescriptor`

[GPUDeviceDescriptor](https://gpuweb.github.io#gpudevicedescriptor)

dictionary GPUDeviceDescriptor GPUObjectDescriptorBase sequence GPUFeatureName requiredFeatures record DOMString GPUSize64 or undefined requiredLimits GPUQueueDescriptor defaultQueue 

[GPUDeviceDescriptor](https://gpuweb.github.io#gpudevicedescriptor)

- `requiredFeatures`,  of type sequence\<[GPUFeatureName](https://gpuweb.github.io#gpufeaturename)>, defaulting to `[]`
- Specifies the [features](https://gpuweb.github.io#feature) that are required by the device request.
The request will fail if the adapter cannot provide these features.
Exactly the specified set of features, and no more or less, will be allowed in validation of API calls on the resulting device.
- `requiredLimits`,  of type `record<DOMString, (GPUSize64 or undefined)>`, defaulting to `{}`
- Specifies the [limits](https://gpuweb.github.io#limit) that are required by the device request.
The request will fail if the adapter cannot provide these limits.
Each key with a non-`undefined` value must be the name of a member of [supported limits](https://gpuweb.github.io#supported-limits).
API calls on the resulting device perform validation according to the exact limits of the
device (not the adapter; see [§ 3.6.2 Limits](https://gpuweb.github.io#limits)).
- `defaultQueue`,  of type [GPUQueueDescriptor](https://gpuweb.github.io#gpuqueuedescriptor), defaulting to `{}`
- The descriptor for the default [GPUQueue](https://gpuweb.github.io#gpuqueue)

[Requesting a](https://gpuweb.github.io#example-8a22c3e3)

[GPUDevice](https://gpuweb.github.io#gpudevice)["texture-compression-astc"](https://gpuweb.github.io#texture-compression-astc)
const gpuAdapter= await navigator. gpu. requestAdapter(); const requiredFeatures= \[\]; if ( gpuAdapter. features. has( 'texture-compression-astc' )) { requiredFeatures. push( 'texture-compression-astc' ) } const gpuDevice= await gpuAdapter. requestDevice({ requiredFeatures}); 

[Requesting a](https://gpuweb.github.io#example-f8cb4961)

[GPUDevice](https://gpuweb.github.io#gpudevice)[maxColorAttachmentBytesPerSample](https://gpuweb.github.io#dom-supported-limits-maxcolorattachmentbytespersample)
const gpuAdapter= await navigator. gpu. requestAdapter(); if ( gpuAdapter. limits. maxColorAttachmentBytesPerSample\< 64 ) { // When the desired limit isn't supported, take action to either fall back to a code // path that does not require the higher limit or notify the user that their device // does not meet minimum requirements. } // Request higher limit of max color attachments bytes per sample. const gpuDevice= await gpuAdapter. requestDevice({ requiredLimits: { maxColorAttachmentBytesPerSample: 64 }, }); 

### 4.4. `GPUDevice`

A [GPUDevice](https://gpuweb.github.io#gpudevice)[device](https://gpuweb.github.io#device) and exposes
the functionality of that device.

[GPUDevice](https://gpuweb.github.io#gpudevice)[WebGPU interfaces](https://gpuweb.github.io#webgpu-interface) are created.

To get a [GPUDevice](https://gpuweb.github.io#gpudevice)[requestDevice()](https://gpuweb.github.io#dom-gpuadapter-requestdevice)

\[Exposed Window ,Worker ),SecureContext interface GPUDevice EventTarget SameObject readonly attribute GPUSupportedFeatures features SameObject readonly attribute GPUSupportedLimits limits SameObject readonly attribute GPUAdapterInfo adapterInfo SameObject readonly attribute GPUQueue queue undefined destroy GPUBuffer createBuffer GPUBufferDescriptor descriptor GPUTexture createTexture GPUTextureDescriptor descriptor GPUSampler createSampler optional GPUSamplerDescriptor descriptor GPUExternalTexture importExternalTexture GPUExternalTextureDescriptor descriptor GPUBindGroupLayout createBindGroupLayout GPUBindGroupLayoutDescriptor descriptor GPUPipelineLayout createPipelineLayout GPUPipelineLayoutDescriptor descriptor GPUBindGroup createBindGroup GPUBindGroupDescriptor descriptor GPUShaderModule createShaderModule GPUShaderModuleDescriptor descriptor GPUComputePipeline createComputePipeline GPUComputePipelineDescriptor descriptor GPURenderPipeline createRenderPipeline GPURenderPipelineDescriptor descriptor Promise GPUComputePipeline createComputePipelineAsync GPUComputePipelineDescriptor descriptor Promise GPURenderPipeline createRenderPipelineAsync GPURenderPipelineDescriptor descriptor GPUCommandEncoder createCommandEncoder optional GPUCommandEncoderDescriptor descriptor GPURenderBundleEncoder createRenderBundleEncoder GPURenderBundleEncoderDescriptor descriptor GPUQuerySet createQuerySet GPUQuerySetDescriptor descriptor GPUDevice includes GPUObjectBase 

[GPUDevice](https://gpuweb.github.io#gpudevice)[immutable properties](https://gpuweb.github.io#immutable-property):

- `features`,  of type [GPUSupportedFeatures](https://gpuweb.github.io#gpusupportedfeatures), readonly
- A set containing the [GPUFeatureName](https://gpuweb.github.io#gpufeaturename)[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
- `limits`,  of type [GPUSupportedLimits](https://gpuweb.github.io#gpusupportedlimits), readonly
- The limits supported by the device ([\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)
- `queue`,  of type [GPUQueue](https://gpuweb.github.io#gpuqueue), readonly
- The primary [GPUQueue](https://gpuweb.github.io#gpuqueue)
- `adapterInfo`,  of type [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo), readonly
- Information about the physical adapter which created the [device](https://gpuweb.github.io#device) that this [GPUDevice](https://gpuweb.github.io#gpudevice)
For a given [GPUDevice](https://gpuweb.github.io#gpudevice)[GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)
The same object is returned each time. To create that object for the first time: **Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Returns:** [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Return a [new adapter info](https://gpuweb.github.io#abstract-opdef-new-adapter-info) for `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[adapter\]\]](https://gpuweb.github.io#dom-device-adapter-slot)

The [\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[GPUDevice](https://gpuweb.github.io#gpudevice)[device](https://gpuweb.github.io#device) that the [GPUDevice](https://gpuweb.github.io#gpudevice)

[GPUDevice](https://gpuweb.github.io#gpudevice)

- `destroy()`
- Destroys the [device](https://gpuweb.github.io#device), preventing further operations on it.
Outstanding asynchronous operations will fail.
Note: It is valid to destroy a device multiple times. **Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. [Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)[GPUBuffer](https://gpuweb.github.io#gpubuffer)
  2. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  1. [Lose the device](https://gpuweb.github.io#lose-the-device)(`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)["destroyed"](https://gpuweb.github.io#dom-gpudevicelostreason-destroyed)
Note: Since no further operations can be enqueued on this device, implementations can abort outstanding asynchronous operations immediately and free resource allocations, including mapped memory that was just unmapped.

[GPUDevice](https://gpuweb.github.io#gpudevice)[GPUDevice](https://gpuweb.github.io#gpudevice)
- Always allowed:
[COPY\_SRC](https://gpuweb.github.io#dom-gputextureusage-copy_src)[COPY\_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)[TEXTURE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-texture_binding)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[TRANSIENT\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-transient_attachment)

### 4.5. Example

[A more robust example of requesting a](https://gpuweb.github.io#example-2e76e7cd)

[GPUAdapter](https://gpuweb.github.io#gpuadapter)[GPUDevice](https://gpuweb.github.io#gpudevice)
let gpuDevice= null ; async function initializeWebGPU() { // Check to ensure the user agent supports WebGPU. if ( ! ( 'gpu' in navigator)) { console. error( "User agent doesn't support WebGPU." ); return false ; } // Request an adapter. const gpuAdapter= await navigator. gpu. requestAdapter(); // requestAdapter may resolve with null if no suitable adapters are found. if ( ! gpuAdapter) { console. error( 'No WebGPU adapters found.' ); return false ; } // Request a device. // Note that the promise will reject if invalid options are passed to the optional // dictionary. To avoid the promise rejecting always check any features and limits // against the adapters features and limits prior to calling requestDevice(). gpuDevice= await gpuAdapter. requestDevice(); // requestDevice will never return null, but if a valid device request can't be // fulfilled for some reason it may resolve to a device which has already been lost. // Additionally, devices can be lost at any time after creation for a variety of reasons // (ie: browser resource management, driver updates), so it's a good idea to always // handle lost devices gracefully. gpuDevice. lost. then(( info) => { console. error( \`WebGPU device was lost: ${ info. message} \` ); gpuDevice= null ; // Many causes for lost devices are transient, so applications should try getting a // new device once a previous one has been lost unless the loss was caused by the // application intentionally destroying the device. Note that any WebGPU resources // created with the previous device (buffers, textures, etc) will need to be // re-created with the new one. if ( info. reason!= 'destroyed' ) { initializeWebGPU(); } }); onWebGPUInitialized(); return true ; } function onWebGPUInitialized() { // Begin creating WebGPU resources here... } initializeWebGPU(); 

## 5. Buffers

### 5.1. `GPUBuffer`

A [GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUBuffers](https://gpuweb.github.io#gpubuffer)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)

[GPUBuffer](https://gpuweb.github.io#gpubuffer)[createBuffer()](https://gpuweb.github.io#dom-gpudevice-createbuffer)[mappedAtCreation](https://gpuweb.github.io#dom-gpubufferdescriptor-mappedatcreation)

\[Exposed Window ,Worker ),SecureContext interface GPUBuffer readonly attribute GPUSize64Out size readonly attribute GPUFlagsConstant usage readonly attribute GPUBufferMapState mapState Promise undefined mapAsync GPUMapModeFlags mode optional GPUSize64 offset optional GPUSize64 size ArrayBuffer getMappedRange optional GPUSize64 offset optional GPUSize64 size undefined unmap undefined destroy GPUBuffer includes GPUObjectBase enum GPUBufferMapState "unmapped" "pending" "mapped" 

[GPUBuffer](https://gpuweb.github.io#gpubuffer)[immutable properties](https://gpuweb.github.io#immutable-property):

- `size`,  of type [GPUSize64Out](https://gpuweb.github.io#typedefdef-gpusize64out), readonly
- The length of the [GPUBuffer](https://gpuweb.github.io#gpubuffer)
- `usage`,  of type [GPUFlagsConstant](https://gpuweb.github.io#typedefdef-gpuflagsconstant), readonly
- The allowed usages for this [GPUBuffer](https://gpuweb.github.io#gpubuffer)

[GPUBuffer](https://gpuweb.github.io#gpubuffer)[content timeline properties](https://gpuweb.github.io#content-timeline-property):

- `mapState`,  of type [GPUBufferMapState](https://gpuweb.github.io#enumdef-gpubuffermapstate), readonly
- The current `GPUBufferMapState` of the buffer:
  - `"unmapped"`
  - The buffer is not mapped for use by `this`.[getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)
  - `"pending"`
  - A mapping of the buffer has been requested, but is pending.
It may succeed, or fail validation in [mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)
  - `"mapped"`
  - The buffer is mapped and `this`.[getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)
The [getter steps](https://webidl.spec.whatwg.org/#getter-steps) are:
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)`null`,
return ["mapped"](https://gpuweb.github.io#dom-gpubuffermapstate-mapped)
  2. If `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`null`,
return ["pending"](https://gpuweb.github.io#dom-gpubuffermapstate-pending)
  3. Return ["unmapped"](https://gpuweb.github.io#dom-gpubuffermapstate-unmapped)
- `[[pending_map]]`, of type [Promise](https://webidl.spec.whatwg.org/#idl-promise)
`null`, initially `null`
- The [Promise](https://webidl.spec.whatwg.org/#idl-promise)[mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)
There is never more than one pending map, because [mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)
- `[[mapping]]`, of type [active buffer mapping](https://gpuweb.github.io#active-buffer-mapping) or `null`, initially `null`
- Set if and only if the buffer is currently mapped for use by [getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)
An active buffer mapping is a structure with the following fields: 
  - data, of type [Data Block](https://tc39.es/ecma262/#sec-data-blocks)
  - The mapping for this [GPUBuffer](https://gpuweb.github.io#gpubuffer)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)[views](https://gpuweb.github.io#active-buffer-mapping-views).
  - mode, of type [GPUMapModeFlags](https://gpuweb.github.io#typedefdef-gpumapmodeflags)
  - The [GPUMapModeFlags](https://gpuweb.github.io#typedefdef-gpumapmodeflags)[mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[createBuffer()](https://gpuweb.github.io#dom-gpudevice-createbuffer)
  - range, of type tuple \[[unsigned long long](https://webidl.spec.whatwg.org/#idl-unsigned-long-long)
[unsigned long long](https://webidl.spec.whatwg.org/#idl-unsigned-long-long)
  - The range of this [GPUBuffer](https://gpuweb.github.io#gpubuffer)
  - views, of type [list](https://infra.spec.whatwg.org/#list)\<[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)
  - The [ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)[unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)

  To initialize an active buffer mapping with mode`mode` and
    range `range`, run the following [content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `size` be `range`\[1\] - `range`\[0\].
  2. Let `data` be [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [CreateByteDataBlock](https://tc39.es/ecma262/#sec-createbytedatablock)(`size`).
NOTE:This may result in a[RangeError](https://webidl.spec.whatwg.org/#exceptiondef-rangeerror)
    - For any size at which `new ArrayBuffer()` would succeed at a given moment,
this allocation **should** succeed at that moment.
    - For any size at which `new ArrayBuffer()` *deterministically* throws a [RangeError](https://webidl.spec.whatwg.org/#exceptiondef-rangeerror)**should** as well.
  3. Return an [active buffer mapping](https://gpuweb.github.io#active-buffer-mapping) with:

[GPUBuffer](https://gpuweb.github.io#gpubuffer)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[internal state]]`
- The current internal state of the buffer:

#### 5.1.1. `GPUBufferDescriptor`

dictionary GPUBufferDescriptor GPUObjectDescriptorBase required GPUSize64 size required GPUBufferUsageFlags usage boolean mappedAtCreation false ; };

[GPUBufferDescriptor](https://gpuweb.github.io#gpubufferdescriptor)

- `size`,  of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The size of the buffer in bytes.
- `usage`,  of type [GPUBufferUsageFlags](https://gpuweb.github.io#typedefdef-gpubufferusageflags)
- The allowed usages for the buffer.
- `mappedAtCreation`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- If `true` creates the buffer in an already mapped state, allowing [getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)[mappedAtCreation](https://gpuweb.github.io#dom-gpubufferdescriptor-mappedatcreation)`true` even if [usage](https://gpuweb.github.io#dom-gpubufferdescriptor-usage)[MAP\_READ](https://gpuweb.github.io#dom-gpubufferusage-map_read)[MAP\_WRITE](https://gpuweb.github.io#dom-gpubufferusage-map_write)
Guarantees that even if the buffer creation eventually fails, it will still appear as if the mapped range can be written/read to until it is unmapped.

#### 5.1.2. Buffer Usages

typedef \[EnforceRange unsigned long GPUBufferUsageFlags Exposed Window ,Worker ),SecureContext namespace GPUBufferUsage const GPUFlagsConstant MAP\_READ const GPUFlagsConstant MAP\_WRITE const GPUFlagsConstant COPY\_SRC const GPUFlagsConstant COPY\_DST const GPUFlagsConstant INDEX const GPUFlagsConstant VERTEX const GPUFlagsConstant UNIFORM const GPUFlagsConstant STORAGE const GPUFlagsConstant INDIRECT const GPUFlagsConstant QUERY\_RESOLVE 

The [GPUBufferUsage](https://gpuweb.github.io#namespacedef-gpubufferusage)[GPUBuffer](https://gpuweb.github.io#gpubuffer)

- `MAP_READ`
- The buffer can be mapped for reading. (Example: calling [mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[GPUMapMode.READ](https://gpuweb.github.io#dom-gpumapmode-read)
May only be combined with [COPY\_DST](https://gpuweb.github.io#dom-gpubufferusage-copy_dst)
- `MAP_WRITE`
- The buffer can be mapped for writing. (Example: calling [mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[GPUMapMode.WRITE](https://gpuweb.github.io#dom-gpumapmode-write)
May only be combined with [COPY\_SRC](https://gpuweb.github.io#dom-gpubufferusage-copy_src)
- `COPY_SRC`
- The buffer can be used as the source of a copy operation. (Examples: as the `source` argument of a [copyBufferToBuffer()](https://gpuweb.github.io#gpucommandencoder-copybuffertobuffer) or [copyBufferToTexture()](https://gpuweb.github.io#dom-gpucommandencoder-copybuffertotexture)
- `COPY_DST`
- The buffer can be used as the destination of a copy or write operation. (Examples: as the
`destination` argument of a [copyBufferToBuffer()](https://gpuweb.github.io#gpucommandencoder-copybuffertobuffer) or [copyTextureToBuffer()](https://gpuweb.github.io#dom-gpucommandencoder-copytexturetobuffer)[writeBuffer()](https://gpuweb.github.io#dom-gpuqueue-writebuffer)
- `INDEX`
- The buffer can be used as an index buffer. (Example: passed to
[setIndexBuffer()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setindexbuffer)
- `VERTEX`
- The buffer can be used as a vertex buffer. (Example: passed to
[setVertexBuffer()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setvertexbuffer)
- `UNIFORM`
- The buffer can be used as a uniform buffer. (Example: as a bind group entry for a
[GPUBufferBindingLayout](https://gpuweb.github.io#dictdef-gpubufferbindinglayout)[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform)
- `STORAGE`
- The buffer can be used as a storage buffer. (Example: as a bind group entry for a
[GPUBufferBindingLayout](https://gpuweb.github.io#dictdef-gpubufferbindinglayout)[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage)
- `INDIRECT`
- The buffer can be used as to store indirect command arguments. (Examples: as the
`indirectBuffer` argument of a [drawIndirect()](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindirect)[dispatchWorkgroupsIndirect()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroupsindirect)
- `QUERY_RESOLVE`
- The buffer can be used to capture query results. (Example: as the `destination` argument of
a [resolveQuerySet()](https://gpuweb.github.io#dom-gpucommandencoder-resolvequeryset)

#### 5.1.3. Buffer Creation

- `createBuffer(descriptor)`
- Creates a [GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createBuffer(descriptor)](https://gpuweb.github.io#dom-gpudevice-createbuffer) method.Parameter Type Nullable Optional Description `descriptor`
[GPUBufferDescriptor](https://gpuweb.github.io#gpubufferdescriptor)
✘ ✘ Description of the [GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Returns:** [GPUBuffer](https://gpuweb.github.io#gpubuffer)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `b` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUBuffer](https://gpuweb.github.io#gpubuffer)`descriptor`).
  2. Set `b`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`descriptor`.[size](https://gpuweb.github.io#dom-gpubufferdescriptor-size)
  3. Set `b`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)`descriptor`.[usage](https://gpuweb.github.io#dom-gpubufferdescriptor-usage)
  4. If `descriptor`.[mappedAtCreation](https://gpuweb.github.io#dom-gpubufferdescriptor-mappedatcreation)`true`:
    1. If `descriptor`.[size](https://gpuweb.github.io#dom-gpubufferdescriptor-size)[RangeError](https://webidl.spec.whatwg.org/#exceptiondef-rangeerror)
    2. Set `b`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [initialize an active buffer mapping](https://gpuweb.github.io#abstract-opdef-initialize-an-active-buffer-mapping) with mode [WRITE](https://gpuweb.github.io#dom-gpumapmode-write)`[0,` .`descriptor`.[size](https://gpuweb.github.io#dom-gpubufferdescriptor-size)
  5. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  6. Return `b`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. If any of the following requirements are unmet,
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `b` and return.
    - `this` must not be [lost](https://gpuweb.github.io#abstract-opdef-invalid).
    - `descriptor`.[usage](https://gpuweb.github.io#dom-gpubufferdescriptor-usage)
    - `descriptor`.[usage](https://gpuweb.github.io#dom-gpubufferdescriptor-usage)[allowed buffer usages](https://gpuweb.github.io#allowed-buffer-usages) for `this`.
    - If `descriptor`.[usage](https://gpuweb.github.io#dom-gpubufferdescriptor-usage)[MAP\_READ](https://gpuweb.github.io#dom-gpubufferusage-map_read)
    - If `descriptor`.[usage](https://gpuweb.github.io#dom-gpubufferdescriptor-usage)[MAP\_WRITE](https://gpuweb.github.io#dom-gpubufferusage-map_write)
    - If `descriptor`.[size](https://gpuweb.github.io#dom-gpubufferdescriptor-size)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxBufferSize](https://gpuweb.github.io#dom-supported-limits-maxbuffersize)
Note: If buffer creation fails, and `descriptor`.[mappedAtCreation](https://gpuweb.github.io#dom-gpubufferdescriptor-mappedatcreation)`false`,
        any calls to [mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)
  1. If `descriptor`.[mappedAtCreation](https://gpuweb.github.io#dom-gpubufferdescriptor-mappedatcreation)`true`:
    1. Set `b`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[unavailable](https://gpuweb.github.io#gpubuffer-internal-state-unavailable)".
Otherwise: 
    1. Set `b`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[available](https://gpuweb.github.io#gpubuffer-internal-state-available)".
  2. Create a device allocation for `b` where each byte is zero.
If the allocation fails without side-effects,
[generate an out-of-memory error](https://gpuweb.github.io#abstract-opdef-generate-an-out-of-memory-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `b`, and return.

[Creating a 128 byte uniform buffer that can be written into:](https://gpuweb.github.io#example-6a51d29e)

const buffer= gpuDevice. createBuffer({ size: 128 , usage: GPUBufferUsage. UNIFORM|GPUBufferUsage. COPY\_DST}); 

#### 5.1.4. Buffer Destruction

An application that no longer requires a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[destroy()](https://gpuweb.github.io#dom-gpubuffer-destroy)

Note: This allows the user agent to reclaim the GPU memory associated with the [GPUBuffer](https://gpuweb.github.io#gpubuffer)

[GPUBuffer](https://gpuweb.github.io#gpubuffer)

- `destroy()`
- Destroys the [GPUBuffer](https://gpuweb.github.io#gpubuffer)
Note: It is valid to destroy a buffer multiple times. **Called on:** [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`this`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Call `this`.[unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)
  2. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Set `this`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[destroyed](https://gpuweb.github.io#gpubuffer-internal-state-destroyed)".
Note: Since no further operations can be enqueued using this buffer, implementations can free resource allocations, including mapped memory that was just unmapped.

### 5.2. Buffer Mapping

An application can request to map a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)[Queue timeline](https://gpuweb.github.io#queue-timeline).

Once the [GPUBuffer](https://gpuweb.github.io#gpubuffer)[getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[detached](https://webidl.spec.whatwg.org/#dfn-detach) by [unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[destroy()](https://gpuweb.github.io#dom-gpubuffer-destroy)[GPUDevice](https://gpuweb.github.io#gpudevice)[destroy()](https://gpuweb.github.io#dom-gpudevice-destroy)[transferred](https://webidl.spec.whatwg.org/#arraybuffer-transfer).
A [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)

typedef \[EnforceRange unsigned long GPUMapModeFlags Exposed Window ,Worker ),SecureContext namespace GPUMapMode const GPUFlagsConstant READ const GPUFlagsConstant WRITE 

The [GPUMapMode](https://gpuweb.github.io#namespacedef-gpumapmode)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)

- `READ`
- Only valid with buffers created with the [MAP\_READ](https://gpuweb.github.io#dom-gpubufferusage-map_read)
Once the buffer is mapped, calls to [getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)
- `WRITE`
- Only valid with buffers created with the [MAP\_WRITE](https://gpuweb.github.io#dom-gpubufferusage-map_write)
Once the buffer is mapped, calls to [getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)
Note: Since the [MAP\_WRITE](https://gpuweb.github.io#dom-gpubufferusage-map_write)[COPY\_SRC](https://gpuweb.github.io#dom-gpubufferusage-copy_src)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)

[GPUBuffer](https://gpuweb.github.io#gpubuffer)

- `mapAsync(mode, offset, size)`
- Maps the given range of the [GPUBuffer](https://gpuweb.github.io#gpubuffer)[Promise](https://webidl.spec.whatwg.org/#idl-promise)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)
The resolution of the returned [Promise](https://webidl.spec.whatwg.org/#idl-promise)**only** indicates that the buffer has been mapped.
It does not guarantee the completion of any other operations visible to the [content timeline](https://gpuweb.github.io#content-timeline),
and in particular does not imply that any other [Promise](https://webidl.spec.whatwg.org/#idl-promise)[onSubmittedWorkDone()](https://gpuweb.github.io#dom-gpuqueue-onsubmittedworkdone)[mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[GPUBuffer](https://gpuweb.github.io#gpubuffer)
The resolution of the [Promise](https://webidl.spec.whatwg.org/#idl-promise)[onSubmittedWorkDone()](https://gpuweb.github.io#dom-gpuqueue-onsubmittedworkdone)**does** imply the completion of [mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`this`. **Arguments:**
Arguments for the [GPUBuffer.mapAsync(mode, offset, size)](https://gpuweb.github.io#dom-gpubuffer-mapasync) method.Parameter Type Nullable Optional Description `mode`
[GPUMapModeFlags](https://gpuweb.github.io#typedefdef-gpumapmodeflags)
✘ ✘ Whether the buffer should be mapped for reading or writing. `offset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Offset in bytes into the buffer to the start of the range to map. `size`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Size in bytes of the range to map. [Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `contentTimeline` be the current [Content timeline](https://gpuweb.github.io#content-timeline).
  2. If `this`.[mapState](https://gpuweb.github.io#dom-gpubuffer-mapstate)["unmapped"](https://gpuweb.github.io#dom-gpubuffermapstate-unmapped)
    1. Issue the `early-reject steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
    2. Return [a promise rejected with](https://webidl.spec.whatwg.org/#a-promise-rejected-with) [OperationError](https://webidl.spec.whatwg.org/#operationerror)
  3. Let `p` be a new [Promise](https://webidl.spec.whatwg.org/#idl-promise)
  4. Set `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`p`.
  5. Issue the `validation steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  6. Return `p`.
[Device timeline](https://gpuweb.github.io#device-timeline) `validation steps`:
  1. If `size` is `undefined`:
    1. Let `rangeSize` be max(0, `this`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`offset`).
Otherwise: 
    1. Let `rangeSize` be `size`.
  2. If any of the following conditions are unsatisfied: 
    - `this` must be [valid](https://gpuweb.github.io#abstract-opdef-valid).
    1. Set `deviceLost` to `true`.
    2. Issue the `map failure steps` on `contentTimeline`.
    3. Return.
  3. If any of the following conditions are unsatisfied: 
    - `this`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[available](https://gpuweb.github.io#gpubuffer-internal-state-available)".
    - `offset` is a multiple of 8.
    - `rangeSize` is a multiple of 4.
    - `offset` + `rangeSize` ≤ `this`.[size](https://gpuweb.github.io#dom-gpubuffer-size)
    - `mode` contains only bits defined in [GPUMapMode](https://gpuweb.github.io#namespacedef-gpumapmode)
    - `mode` contains exactly one of [READ](https://gpuweb.github.io#dom-gpumapmode-read)[WRITE](https://gpuweb.github.io#dom-gpumapmode-write)
    - If `mode` contains [READ](https://gpuweb.github.io#dom-gpumapmode-read)`this`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[MAP\_READ](https://gpuweb.github.io#dom-gpubufferusage-map_read)
    - If `mode` contains [WRITE](https://gpuweb.github.io#dom-gpumapmode-write)`this`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[MAP\_WRITE](https://gpuweb.github.io#dom-gpubufferusage-map_write)
Then: 
    1. Set `deviceLost` to `false`.
    2. Issue the `map failure steps` on `contentTimeline`.
    3. [Generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).
    4. Return.
  4. Set `this`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[unavailable](https://gpuweb.github.io#gpubuffer-internal-state-unavailable)".
Note: Since the buffer is mapped, its contents cannot change between this step and [unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)
  5. When either of the following events occur (whichever comes first), or if either has already occurred: 
    - The [device timeline](https://gpuweb.github.io#device-timeline) becomes informed of the completion of an unspecified [queue timeline](https://gpuweb.github.io#queue-timeline) point:
      - after the completion of
currently-enqueued operations that use `this`
      - and no later than the completion of
all currently-enqueued operations
(regardless of whether they use `this`).
    - `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[becomes lost](https://gpuweb.github.io#becomes-lost).
Then issue the subsequent steps on the [device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Set `deviceLost` to `true` if `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[lost](https://gpuweb.github.io#abstract-opdef-invalid),
and `false` otherwise.
Note: The device could have been lost between the previous block of steps and this one.
  2. If `deviceLost`:
    1. Issue the `map failure steps` on `contentTimeline`.
Otherwise: 
    1. Let `internalStateAtCompletion` be `this`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)
Note: If, and only if, at this point the buffer has become "[available](https://gpuweb.github.io#gpubuffer-internal-state-available)"
again due to an [unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`p` below,
so mapping will not succeed in the steps below.
    2. Let `dataForMappedRegion` be the contents of `this` starting at offset `offset`, for `rangeSize` bytes.
    3. Issue the `map success steps` on the `contentTimeline`.
[Content timeline](https://gpuweb.github.io#content-timeline) `map success steps`:
  1. If `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`p`:
Note: The map has been cancelled by [unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)
    1. [Assert](https://infra.spec.whatwg.org/#assert) `p` is rejected.
    2. Return.
  2. [Assert](https://infra.spec.whatwg.org/#assert) `p` is pending.
  3. [Assert](https://infra.spec.whatwg.org/#assert) `internalStateAtCompletion` is "[unavailable](https://gpuweb.github.io#gpubuffer-internal-state-unavailable)".
  4. Let `mapping` be [initialize an active buffer mapping](https://gpuweb.github.io#abstract-opdef-initialize-an-active-buffer-mapping) with mode `mode` and range `[`.`offset`, `offset` + `rangeSize`\]
If this allocation fails: 
    1. Set `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`null`,
and [reject](https://webidl.spec.whatwg.org/#reject) `p` with a [RangeError](https://webidl.spec.whatwg.org/#exceptiondef-rangeerror)
    2. Return.
  5. Set the content of `mapping`.[data](https://gpuweb.github.io#active-buffer-mapping-data) to `dataForMappedRegion`.
  6. Set `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)`mapping`.
  7. Set `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`null`,
and [resolve](https://webidl.spec.whatwg.org/#resolve) `p`.
[Content timeline](https://gpuweb.github.io#content-timeline) `map failure steps`:
  1. If `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`p`:
Note: The map has been cancelled by [unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)
    1. [Assert](https://infra.spec.whatwg.org/#assert) `p` is already rejected.
    2. Return.
  2. [Assert](https://infra.spec.whatwg.org/#assert) `p` is still pending.
  3. Set `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`null`.
  4. If `deviceLost`:
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `p` with an [AbortError](https://webidl.spec.whatwg.org/#aborterror)
Note: This is the same error type produced by cancelling the map using
[unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)
Otherwise: 
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `p` with an [OperationError](https://webidl.spec.whatwg.org/#operationerror)
- `getMappedRange(offset, size)`
- Returns an [ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`this`. **Arguments:**
Arguments for the [GPUBuffer.getMappedRange(offset, size)](https://gpuweb.github.io#dom-gpubuffer-getmappedrange) method.Parameter Type Nullable Optional Description `offset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Offset in bytes into the buffer to return buffer contents from. `size`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Size in bytes of the [ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)
**Returns:** [ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If `size` is missing:
    1. Let `rangeSize` be max(0, `this`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`offset`).
Otherwise, let `rangeSize` be `size`.
  2. If any of the following conditions are unsatisfied, throw an [OperationError](https://webidl.spec.whatwg.org/#operationerror)
    - `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)`null`.
    - `offset` is a multiple of 8.
    - `rangeSize` is a multiple of 4.
    - `offset` ≥ `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[range](https://gpuweb.github.io#active-buffer-mapping-range)\[0\].
    - `offset` + `rangeSize` ≤ `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[range](https://gpuweb.github.io#active-buffer-mapping-range)\[1\].
    - \[`offset`, `offset` + `rangeSize`) does not overlap another range in `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[views](https://gpuweb.github.io#active-buffer-mapping-views).
Note: It is always valid to get mapped ranges of a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[mappedAtCreation](https://gpuweb.github.io#dom-gpubufferdescriptor-mappedatcreation)[invalid](https://gpuweb.github.io#abstract-opdef-invalid), because
    the [Content timeline](https://gpuweb.github.io#content-timeline) might not know it is invalid.
  3. Let `data` be `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[data](https://gpuweb.github.io#active-buffer-mapping-data).
  4. Let `view` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create an ArrayBuffer](https://webidl.spec.whatwg.org/#arraybuffer-create) of size `rangeSize`,
but with its pointer mutably referencing the content of `data` at offset
(`offset` - [\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[range](https://gpuweb.github.io#active-buffer-mapping-range)\[0\]).
Note: A [RangeError](https://webidl.spec.whatwg.org/#exceptiondef-rangeerror)`data` has already
been allocated during [mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[createBuffer()](https://gpuweb.github.io#dom-gpudevice-createbuffer)
  5. Set `view`.[\[\[ArrayBufferDetachKey\]\]](https://tc39.es/ecma262/#sec-properties-of-the-arraybuffer-instances)
Note: This causes a [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)[DetachArrayBuffer](https://tc39.es/ecma262/#sec-detacharraybuffer), except by [unmap()](https://gpuweb.github.io#dom-gpubuffer-unmap)
  6. [Append](https://infra.spec.whatwg.org/#list-append) `view` to `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[views](https://gpuweb.github.io#active-buffer-mapping-views).
  7. Return `view`.
Note: User agents should consider issuing a developer-visible warning if
        [getMappedRange()](https://gpuweb.github.io#dom-gpubuffer-getmappedrange)[mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[mapState](https://gpuweb.github.io#dom-gpubuffer-mapstate)["mapped"](https://gpuweb.github.io#dom-gpubuffermapstate-mapped)[onSubmittedWorkDone()](https://gpuweb.github.io#dom-gpuqueue-onsubmittedworkdone)
- `unmap()`
- Unmaps the mapped range of the [GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`this`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`null`:
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)[AbortError](https://webidl.spec.whatwg.org/#aborterror)
    2. Set `this`.[\[\[pending\_map\]\]](https://gpuweb.github.io#dom-gpubuffer-pending_map-slot)`null`.
  2. If `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)`null`:
    1. Return.
  3. For each [ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)`ab` in `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[views](https://gpuweb.github.io#active-buffer-mapping-views):
    1. Perform [DetachArrayBuffer](https://tc39.es/ecma262/#sec-detacharraybuffer)(`ab`, "WebGPUBufferMapping").
  4. Let `bufferUpdate` be `null`.
  5. If `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[mode](https://gpuweb.github.io#active-buffer-mapping-mode) contains [WRITE](https://gpuweb.github.io#dom-gpumapmode-write)
    1. Set `bufferUpdate` to { `data`: `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[data](https://gpuweb.github.io#active-buffer-mapping-data), `offset`: `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)[range](https://gpuweb.github.io#active-buffer-mapping-range)\[0\]
}.
Note: When a buffer is mapped without the [WRITE](https://gpuweb.github.io#dom-gpumapmode-write)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)
  6. Set `this`.[\[\[mapping\]\]](https://gpuweb.github.io#dom-gpubuffer-mapping-slot)`null`.
  7. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. If any of the following conditions are unsatisfied, return. 
    - `this` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  2. [Assert](https://infra.spec.whatwg.org/#assert) `this`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[unavailable](https://gpuweb.github.io#gpubuffer-internal-state-unavailable)".
  3. If `bufferUpdate` is not `null`:
    1. Issue the following steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[queue](https://gpuweb.github.io#dom-gpudevice-queue)
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
      1. Update the contents of `this` at offset `bufferUpdate`.`offset` with the data `bufferUpdate`.`data`.
  4. Set `this`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[available](https://gpuweb.github.io#gpubuffer-internal-state-available)".

## 6. Textures and Texture Views

### 6.1. `GPUTexture`

A texture is made up of [1d](https://gpuweb.github.io#dom-gputexturedimension-1d)[2d](https://gpuweb.github.io#dom-gputexturedimension-2d)[3d](https://gpuweb.github.io#dom-gputexturedimension-3d)[GPUTextureUsage](https://gpuweb.github.io#namespacedef-gputextureusage)

One [texture](https://gpuweb.github.io#texture) consists of one or more texture subresources,
each uniquely identified by a [mipmap level](https://gpuweb.github.io#mipmap-level) and,
for [2d](https://gpuweb.github.io#dom-gputexturedimension-2d)[array layer](https://gpuweb.github.io#array-layer) and [aspect](https://gpuweb.github.io#aspect).

A [texture subresource](https://gpuweb.github.io#texture-subresources) is a [subresource](https://gpuweb.github.io#subresource): each can be used in different
[internal usages](https://gpuweb.github.io#internal-usage) within a single [usage scope](https://gpuweb.github.io#usage-scope).

Each subresource in a mipmap level is approximately half the size,
in each spatial dimension, of the corresponding resource in the lesser level
(see [logical miplevel-specific texture extent](https://gpuweb.github.io#logical-miplevel-specific-texture-extent)).
The subresource in level 0 has the dimensions of the texture itself.
Smaller levels are typically used to store lower resolution versions of the same image.
[GPUSampler](https://gpuweb.github.io#gpusampler)[levels of detail](https://gpuweb.github.io#levels-of-detail), explicitly or automatically.

A ["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)

Each subresource has an aspect.
Color textures have just one aspect: color.
[Depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format) textures may have multiple aspects:
a depth aspect,
a stencil aspect, or both, and may be used in special ways, such as in
[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth)

A ["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)`z` value in the texture.
Slices are not separate subresources.

\[Exposed Window ,Worker ),SecureContext interface GPUTexture GPUTextureView createView optional GPUTextureViewDescriptor descriptor undefined destroy readonly attribute GPUIntegerCoordinateOut width readonly attribute GPUIntegerCoordinateOut height readonly attribute GPUIntegerCoordinateOut depthOrArrayLayers readonly attribute GPUIntegerCoordinateOut mipLevelCount readonly attribute GPUSize32Out sampleCount readonly attribute GPUTextureDimension dimension readonly attribute GPUTextureFormat format readonly attribute GPUFlagsConstant usage readonly attribute (GPUTextureViewDimension or undefined textureBindingViewDimension GPUTexture includes GPUObjectBase 

[GPUTexture](https://gpuweb.github.io#gputexture)[immutable properties](https://gpuweb.github.io#immutable-property):

- `width`,  of type [GPUIntegerCoordinateOut](https://gpuweb.github.io#typedefdef-gpuintegercoordinateout), readonly
- The width of this [GPUTexture](https://gpuweb.github.io#gputexture)
- `height`,  of type [GPUIntegerCoordinateOut](https://gpuweb.github.io#typedefdef-gpuintegercoordinateout), readonly
- The height of this [GPUTexture](https://gpuweb.github.io#gputexture)
- `depthOrArrayLayers`,  of type [GPUIntegerCoordinateOut](https://gpuweb.github.io#typedefdef-gpuintegercoordinateout), readonly
- The depth or layer count of this [GPUTexture](https://gpuweb.github.io#gputexture)
- `mipLevelCount`,  of type [GPUIntegerCoordinateOut](https://gpuweb.github.io#typedefdef-gpuintegercoordinateout), readonly
- The number of mip levels of this [GPUTexture](https://gpuweb.github.io#gputexture)
- `sampleCount`,  of type [GPUSize32Out](https://gpuweb.github.io#typedefdef-gpusize32out), readonly
- The number of sample count of this [GPUTexture](https://gpuweb.github.io#gputexture)
- `dimension`,  of type [GPUTextureDimension](https://gpuweb.github.io#enumdef-gputexturedimension), readonly
- The dimension of the set of texel for each of this [GPUTexture](https://gpuweb.github.io#gputexture)
- `format`,  of type [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat), readonly
- The format of this [GPUTexture](https://gpuweb.github.io#gputexture)
- `usage`,  of type [GPUFlagsConstant](https://gpuweb.github.io#typedefdef-gpuflagsconstant), readonly
- The allowed usages for this [GPUTexture](https://gpuweb.github.io#gputexture)
- `[[viewFormats]]`, of type [sequence](https://webidl.spec.whatwg.org/#idl-sequence)\<[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- The set of [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[GPUTextureViewDescriptor](https://gpuweb.github.io#dictdef-gputextureviewdescriptor)[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)[GPUTexture](https://gpuweb.github.io#gputexture)
- `textureBindingViewDimension`,  of type `(GPUTextureViewDimension or undefined)`, readonly
- 
     On devices without["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
On devices with ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)`undefined`, and there is no such restriction.

[GPUTexture](https://gpuweb.github.io#gputexture)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[destroyed]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
`false`
- If the texture is destroyed, it can no longer be used in any operation, and its underlying memory can be freed.

**Arguments:**

- [GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)`baseSize`
- [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)`mipLevel`

**Returns:** [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `extent` be a new [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)
2. Set `extent`.[width](https://gpuweb.github.io#dom-gpuextent3ddict-width)`baseSize`.[width](https://gpuweb.github.io#gpuextent3d-width) ≫ `mipLevel`).
3. Set `extent`.[height](https://gpuweb.github.io#dom-gpuextent3ddict-height)`baseSize`.[height](https://gpuweb.github.io#gpuextent3d-height) ≫ `mipLevel`).
4. Set `extent`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gpuextent3ddict-depthorarraylayers)
5. Return `extent`.

The logical miplevel-specific texture extent of a [texture](https://gpuweb.github.io#texture) is the size of the
[texture](https://gpuweb.github.io#texture) in texels at a specific miplevel. It is calculated by this procedure:

**Arguments:**

- [GPUTextureDescriptor](https://gpuweb.github.io#gputexturedescriptor)`descriptor`
- [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)`mipLevel`

**Returns:** [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)

1. Let `extent` be a new [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)
2. If `descriptor`.[dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension)
  - ["1d"](https://gpuweb.github.io#dom-gputexturedimension-1d)
    - Set `extent`.[width](https://gpuweb.github.io#dom-gpuextent3ddict-width)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width) ≫ `mipLevel`).
    - Set `extent`.[height](https://gpuweb.github.io#dom-gpuextent3ddict-height)
    - Set `extent`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gpuextent3ddict-depthorarraylayers)
  - ["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
    - Set `extent`.[width](https://gpuweb.github.io#dom-gpuextent3ddict-width)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width) ≫ `mipLevel`).
    - Set `extent`.[height](https://gpuweb.github.io#dom-gpuextent3ddict-height)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[height](https://gpuweb.github.io#gpuextent3d-height) ≫ `mipLevel`).
    - Set `extent`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gpuextent3ddict-depthorarraylayers)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers).
  - ["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)
    - Set `extent`.[width](https://gpuweb.github.io#dom-gpuextent3ddict-width)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width) ≫ `mipLevel`).
    - Set `extent`.[height](https://gpuweb.github.io#dom-gpuextent3ddict-height)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[height](https://gpuweb.github.io#gpuextent3d-height) ≫ `mipLevel`).
    - Set `extent`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gpuextent3ddict-depthorarraylayers)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) ≫ `mipLevel`).
3. Return `extent`.

The physical miplevel-specific texture extent of a [texture](https://gpuweb.github.io#texture) is the size of the
[texture](https://gpuweb.github.io#texture) in texels at a specific miplevel that includes the possible extra padding
to form complete [texel blocks](https://gpuweb.github.io#texel-block) in the [texture](https://gpuweb.github.io#texture). It is calculated by this procedure:

**Arguments:**

- [GPUTextureDescriptor](https://gpuweb.github.io#gputexturedescriptor)`descriptor`
- [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)`mipLevel`

**Returns:** [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)

1. Let `extent` be a new [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)
2. Let `logicalExtent` be [logical miplevel-specific texture extent](https://gpuweb.github.io#logical-miplevel-specific-texture-extent)(`descriptor`, `mipLevel`).
3. If `descriptor`.[dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension)
  - ["1d"](https://gpuweb.github.io#dom-gputexturedimension-1d)
    - Set `extent`.[width](https://gpuweb.github.io#dom-gpuextent3ddict-width)`logicalExtent`.[width](https://gpuweb.github.io#gpuextent3d-width) rounded up to the nearest multiple of `descriptor`’s [texel block width](https://gpuweb.github.io#texel-block-width).
    - Set `extent`.[height](https://gpuweb.github.io#dom-gpuextent3ddict-height)
    - Set `extent`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gpuextent3ddict-depthorarraylayers)
  - ["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
    - Set `extent`.[width](https://gpuweb.github.io#dom-gpuextent3ddict-width)`logicalExtent`.[width](https://gpuweb.github.io#gpuextent3d-width) rounded up to the nearest multiple of `descriptor`’s [texel block width](https://gpuweb.github.io#texel-block-width).
    - Set `extent`.[height](https://gpuweb.github.io#dom-gpuextent3ddict-height)`logicalExtent`.[height](https://gpuweb.github.io#gpuextent3d-height) rounded up to the nearest multiple of `descriptor`’s [texel block height](https://gpuweb.github.io#texel-block-height).
    - Set `extent`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gpuextent3ddict-depthorarraylayers)`logicalExtent`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers).
  - ["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)
    - Set `extent`.[width](https://gpuweb.github.io#dom-gpuextent3ddict-width)`logicalExtent`.[width](https://gpuweb.github.io#gpuextent3d-width) rounded up to the nearest multiple of `descriptor`’s [texel block width](https://gpuweb.github.io#texel-block-width).
    - Set `extent`.[height](https://gpuweb.github.io#dom-gpuextent3ddict-height)`logicalExtent`.[height](https://gpuweb.github.io#gpuextent3d-height) rounded up to the nearest multiple of `descriptor`’s [texel block height](https://gpuweb.github.io#texel-block-height).
    - Set `extent`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gpuextent3ddict-depthorarraylayers)`logicalExtent`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers).
4. Return `extent`.

#### 6.1.1. `GPUTextureDescriptor`

dictionary GPUTextureDescriptor GPUObjectDescriptorBase required GPUExtent3D size GPUIntegerCoordinate mipLevelCount GPUSize32 sampleCount GPUTextureDimension dimension required GPUTextureFormat format required GPUTextureUsageFlags usage sequence GPUTextureFormat viewFormats GPUTextureViewDimension textureBindingViewDimension 

[GPUTextureDescriptor](https://gpuweb.github.io#gputexturedescriptor)

- `size`,  of type [GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
- The width, height, and depth or layer count of the texture.
- `mipLevelCount`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate), defaulting to `1`
- The number of mip levels the texture will contain.
- `sampleCount`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32), defaulting to `1`
- The sample count of the texture. A [sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)`1` indicates
a multisampled texture.
- `dimension`,  of type [GPUTextureDimension](https://gpuweb.github.io#enumdef-gputexturedimension), defaulting to `"2d"`
- Whether the texture is one-dimensional, an array of two-dimensional layers, or three-dimensional.
- `format`,  of type [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- The format of the texture.
- `usage`,  of type [GPUTextureUsageFlags](https://gpuweb.github.io#typedefdef-gputextureusageflags)
- The allowed usages for the texture.
- `viewFormats`,  of type sequence\<[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)>, defaulting to `[]`
- Specifies what view [format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)[createView()](https://gpuweb.github.io#dom-gputexture-createview)[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)
NOTE:Adding a format to this list may have a significant performance impact, so it is best to avoid adding formats unnecessarily. The actual performance impact is highly dependent on the target system; developers must
    test various systems to find out the impact on their particular application.
    For example, on some systems any texture with a [format](https://gpuweb.github.io#dom-gputexturedescriptor-format)[viewFormats](https://gpuweb.github.io#dom-gputexturedescriptor-viewformats)["rgba8unorm-srgb"](https://gpuweb.github.io#dom-gputextureformat-rgba8unorm-srgb)["rgba8unorm"](https://gpuweb.github.io#dom-gputextureformat-rgba8unorm)
Formats in this list must be [texture view format compatible](https://gpuweb.github.io#texture-view-format-compatible) with the texture format.
Two[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
`format` and `viewFormat` are texture view format compatible on a given `device` if:
  - `format` equals `viewFormat`, or
  - `format` and `viewFormat` differ only in whether they are `srgb` formats (have the `-srgb` suffix) and `device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)[contains](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
- `textureBindingViewDimension`,  of type [GPUTextureViewDimension](https://gpuweb.github.io#enumdef-gputextureviewdimension)
- 
     On devices without["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
On devices with ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)

enum GPUTextureDimension "1d" "2d" "3d" 

- `"1d"`
- Specifies a texture that has one dimension, width. ["1d"](https://gpuweb.github.io#dom-gputexturedimension-1d)
- `"2d"`
- Specifies a texture that has a width and height, and may have layers.
- `"3d"`
- Specifies a texture that has a width, height, and depth. ["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)[plain color formats](https://gpuweb.github.io#plain-color-formats) and some [packed/compressed formats](https://gpuweb.github.io#packed-formats)).

#### 6.1.2. Texture Usages

typedef \[EnforceRange unsigned long GPUTextureUsageFlags Exposed Window ,Worker ),SecureContext namespace GPUTextureUsage const GPUFlagsConstant COPY\_SRC const GPUFlagsConstant COPY\_DST const GPUFlagsConstant TEXTURE\_BINDING const GPUFlagsConstant STORAGE\_BINDING const GPUFlagsConstant RENDER\_ATTACHMENT const GPUFlagsConstant TRANSIENT\_ATTACHMENT 

The [GPUTextureUsage](https://gpuweb.github.io#namespacedef-gputextureusage)[GPUTexture](https://gpuweb.github.io#gputexture)

- `COPY_SRC`
- The texture can be used as the source of a copy operation. (Examples: as the `source` argument of a [copyTextureToTexture()](https://gpuweb.github.io#dom-gpucommandencoder-copytexturetotexture)[copyTextureToBuffer()](https://gpuweb.github.io#dom-gpucommandencoder-copytexturetobuffer)
- `COPY_DST`
- The texture can be used as the destination of a copy or write operation. (Examples: as the
`destination` argument of a [copyTextureToTexture()](https://gpuweb.github.io#dom-gpucommandencoder-copytexturetotexture)[copyBufferToTexture()](https://gpuweb.github.io#dom-gpucommandencoder-copybuffertotexture)[writeTexture()](https://gpuweb.github.io#dom-gpuqueue-writetexture)
- `TEXTURE_BINDING`
- The texture can be bound for use as a sampled texture in a shader (Example: as a bind group
entry for a [GPUTextureBindingLayout](https://gpuweb.github.io#dictdef-gputexturebindinglayout)
- `STORAGE_BINDING`
- The texture can be bound for use as a storage texture in a shader (Example: as a bind group
entry for a [GPUStorageTextureBindingLayout](https://gpuweb.github.io#dictdef-gpustoragetexturebindinglayout)
- `RENDER_ATTACHMENT`
- The texture can be used as a color or depth/stencil attachment in a render pass.
(Example: as a [GPURenderPassColorAttachment](https://gpuweb.github.io#dictdef-gpurenderpasscolorattachment)[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[GPURenderPassDepthStencilAttachment](https://gpuweb.github.io#dictdef-gpurenderpassdepthstencilattachment)[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
- `TRANSIENT_ATTACHMENT`
- The texture is intended to be temporary (a hint for optimization), as it is only used within a render pass.

`dimension`,

`size`)

**Arguments:**

- [GPUTextureDimension](https://gpuweb.github.io#enumdef-gputexturedimension)`dimension`
- [GPUTextureDimension](https://gpuweb.github.io#enumdef-gputexturedimension)`size`

#### 6.1.3. Texture Creation

- `createTexture(descriptor)`
- Creates a [GPUTexture](https://gpuweb.github.io#gputexture)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
**Arguments:**
Arguments for the [GPUDevice.createTexture(descriptor)](https://gpuweb.github.io#dom-gpudevice-createtexture) method.Parameter Type Nullable Optional Description `descriptor`
[GPUTextureDescriptor](https://gpuweb.github.io#gputexturedescriptor)
✘ ✘ Description of the [GPUTexture](https://gpuweb.github.io#gputexture)
**Returns:** [GPUTexture](https://gpuweb.github.io#gputexture)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUExtent3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuextent3d-shape)(`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)
  2. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of `descriptor`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  3. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of each element of `descriptor`.[viewFormats](https://gpuweb.github.io#dom-gputexturedescriptor-viewformats)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  4. Let `t` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUTexture](https://gpuweb.github.io#gputexture)`descriptor`).
  5. Set `t`.[width](https://gpuweb.github.io#dom-gputexture-width)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width).
  6. Set `t`.[height](https://gpuweb.github.io#dom-gputexture-height)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[height](https://gpuweb.github.io#gpuextent3d-height).
  7. Set `t`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gputexture-depthorarraylayers)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers).
  8. Set `t`.[mipLevelCount](https://gpuweb.github.io#dom-gputexture-miplevelcount)`descriptor`.[mipLevelCount](https://gpuweb.github.io#dom-gputexturedescriptor-miplevelcount)
  9. Set `t`.[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)`descriptor`.[sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)
  10. Set `t`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)`descriptor`.[dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension)
  11. Set `t`.[format](https://gpuweb.github.io#dom-gputexture-format)`descriptor`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)
  12. Set `t`.[usage](https://gpuweb.github.io#dom-gputexture-usage)`descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)
  13. 
         If`t`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
    1. If `descriptor`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexturedescriptor-texturebindingviewdimension)[provided](https://infra.spec.whatwg.org/#map-exists):
      1. Set `t`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)`descriptor`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexturedescriptor-texturebindingviewdimension)
Otherwise, if `descriptor`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)
      - ["1d"](https://gpuweb.github.io#dom-gputexturedimension-1d)
      - Set `t`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)["1d"](https://gpuweb.github.io#dom-gputextureviewdimension-1d)
      - ["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
      - If the [array layer count](https://gpuweb.github.io#abstract-opdef-array-layer-count) of `t` is 1:
        - Set `t`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)
Otherwise: 
        - Set `t`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)["2d-array"](https://gpuweb.github.io#dom-gputextureviewdimension-2d-array)
      - ["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)
      - Set `t`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
  14. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  15. Return `t`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `t` and return.
    - [validating GPUTextureDescriptor](https://gpuweb.github.io#abstract-opdef-validating-gputexturedescriptor)(`this`, `descriptor`) returns `true`.
  2. Set `t`.[\[\[viewFormats\]\]](https://gpuweb.github.io#dom-gputexture-viewformats-slot)`descriptor`.[viewFormats](https://gpuweb.github.io#dom-gputexturedescriptor-viewformats)
  3. Create a device allocation for `t` where each block has an [equivalent texel representation](https://gpuweb.github.io#equivalent-texel-representation) to a block with a bit representation of zero.
If the allocation fails without side-effects,
[generate an out-of-memory error](https://gpuweb.github.io#abstract-opdef-generate-an-out-of-memory-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `t`, and return.

`this`,

`descriptor`):

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`this`
- [GPUTextureDescriptor](https://gpuweb.github.io#gputexturedescriptor)`descriptor`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `limits` be `this`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)
2. Return `true` if all of the following requirements are met, and `false` otherwise:
  - `this` must not be [lost](https://gpuweb.github.io#abstract-opdef-invalid).
  - `descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)
  - `descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)`this`’s [allowed texture usages](https://gpuweb.github.io#allowed-texture-usages).
  - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width), `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[height](https://gpuweb.github.io#gpuextent3d-height),
and `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be > zero.
  - `descriptor`.[mipLevelCount](https://gpuweb.github.io#dom-gputexturedescriptor-miplevelcount)
  - `descriptor`.[sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)
  - If `descriptor`.[dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension)
    - ["1d"](https://gpuweb.github.io#dom-gputexturedimension-1d)
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width) must be ≤ `limits`.[maxTextureDimension1D](https://gpuweb.github.io#dom-supported-limits-maxtexturedimension1d)
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[height](https://gpuweb.github.io#gpuextent3d-height) must be 1.
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be 1.
      - `descriptor`.[sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)
      - `descriptor`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)[compressed format](https://gpuweb.github.io#compressed-format) or [depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format).
    - ["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width) must be ≤ `limits`.[maxTextureDimension2D](https://gpuweb.github.io#dom-supported-limits-maxtexturedimension2d)
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[height](https://gpuweb.github.io#gpuextent3d-height) must be ≤ `limits`.[maxTextureDimension2D](https://gpuweb.github.io#dom-supported-limits-maxtexturedimension2d)
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be ≤ `limits`.[maxTextureArrayLayers](https://gpuweb.github.io#dom-supported-limits-maxtexturearraylayers)
    - ["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width) must be ≤ `limits`.[maxTextureDimension3D](https://gpuweb.github.io#dom-supported-limits-maxtexturedimension3d)
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[height](https://gpuweb.github.io#gpuextent3d-height) must be ≤ `limits`.[maxTextureDimension3D](https://gpuweb.github.io#dom-supported-limits-maxtexturedimension3d)
      - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be ≤ `limits`.[maxTextureDimension3D](https://gpuweb.github.io#dom-supported-limits-maxtexturedimension3d)
      - `descriptor`.[sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)
      - `descriptor`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)[§ 26.1 Texture Format Capabilities](https://gpuweb.github.io#texture-format-caps).
  - 
         If`this`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
    1. If `descriptor`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexturedescriptor-texturebindingviewdimension)["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)`this`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be 1.
    2. if `descriptor`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexturedescriptor-texturebindingviewdimension)["cube"](https://gpuweb.github.io#dom-gputextureviewdimension-cube)`this`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be 6.
    3. `descriptor`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexturedescriptor-texturebindingviewdimension)["cube-array"](https://gpuweb.github.io#dom-gputextureviewdimension-cube-array)
Note: this validation only applies to a user-specified textureBindingViewDimension. If no value is provided, the texture’s textureBindingViewDimension is set as described in [createTexture()](https://gpuweb.github.io#dom-gpudevice-createtexture)
  - If `this`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)[contain](https://infra.spec.whatwg.org/#list-contain) ["texture-compression-unaligned"](https://gpuweb.github.io#dom-gpufeaturename-texture-compression-unaligned)
    - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[width](https://gpuweb.github.io#gpuextent3d-width) must be multiple of [texel block width](https://gpuweb.github.io#texel-block-width).
    - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[height](https://gpuweb.github.io#gpuextent3d-height) must be multiple of [texel block height](https://gpuweb.github.io#texel-block-height).
  - If `descriptor`.[sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)
    - `descriptor`.[mipLevelCount](https://gpuweb.github.io#dom-gputexturedescriptor-miplevelcount)
    - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be 1.
    - `descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)
    - `descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)
    - `descriptor`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)[§ 26.1 Texture Format Capabilities](https://gpuweb.github.io#texture-format-caps).
  - `descriptor`.[mipLevelCount](https://gpuweb.github.io#dom-gputexturedescriptor-miplevelcount)[maximum mipLevel count](https://gpuweb.github.io#abstract-opdef-maximum-miplevel-count)(`descriptor`.[dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension)`descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)
  - If `descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)
    - `descriptor`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)[renderable format](https://gpuweb.github.io#renderable).
    - `descriptor`.[dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension)["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)
  - If `descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)
    - `descriptor`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)[§ 26.1.1 Plain color formats](https://gpuweb.github.io#plain-color-formats) table
with [STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)
  - If `descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)[TRANSIENT\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-transient_attachment)
    - `descriptor`.[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)[TRANSIENT\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-transient_attachment)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)
    - `descriptor`.[viewFormats](https://gpuweb.github.io#dom-gputexturedescriptor-viewformats)`[]`.
    - `descriptor`.[dimension](https://gpuweb.github.io#dom-gputexturedescriptor-dimension)["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
    - `descriptor`.[mipLevelCount](https://gpuweb.github.io#dom-gputexturedescriptor-miplevelcount)
    - `descriptor`.[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be 1.
  - For each `viewFormat` in `descriptor`.[viewFormats](https://gpuweb.github.io#dom-gputexturedescriptor-viewformats)`descriptor`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)`viewFormat` must be [texture view format compatible](https://gpuweb.github.io#texture-view-format-compatible) on device `this`.
NOTE:Implementations may consider issuing a developer-visible warning if`viewFormat` is not compatible with any of the
    given [usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)
`viewFormat` will be unusable.

[Creating a 16x16, RGBA, 2D texture with one array layer and one mip level:](https://gpuweb.github.io#example-3f852c03)

const texture= gpuDevice. createTexture({ size: { width: 16 , height: 16 }, format: 'rgba8unorm' , usage: GPUTextureUsage. TEXTURE\_BINDING, }); 

#### 6.1.4. Texture Destruction

An application that no longer requires a [GPUTexture](https://gpuweb.github.io#gputexture)[destroy()](https://gpuweb.github.io#dom-gputexture-destroy)

Note: This allows the user agent to reclaim the GPU memory associated with the [GPUTexture](https://gpuweb.github.io#gputexture)

[GPUTexture](https://gpuweb.github.io#gputexture)

- `destroy()`
- Destroys the [GPUTexture](https://gpuweb.github.io#gputexture)
**Called on:** [GPUTexture](https://gpuweb.github.io#gputexture)
`this`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [device timeline](https://gpuweb.github.io#device-timeline).
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Set `this`.[\[\[destroyed\]\]](https://gpuweb.github.io#dom-gputexture-destroyed-slot)

### 6.2. `GPUTextureView`

A [GPUTextureView](https://gpuweb.github.io#gputextureview)[texture subresources](https://gpuweb.github.io#texture-subresources) defined by
a particular [GPUTexture](https://gpuweb.github.io#gputexture)

\[Exposed Window ,Worker ),SecureContext interface GPUTextureView GPUTextureView includes GPUObjectBase 

[GPUTextureView](https://gpuweb.github.io#gputextureview)[immutable properties](https://gpuweb.github.io#immutable-property):

- `[[texture]]`, readonly
- The [GPUTexture](https://gpuweb.github.io#gputexture)
- `[[descriptor]]`, readonly
- The [GPUTextureViewDescriptor](https://gpuweb.github.io#dictdef-gputextureviewdescriptor)
All optional fields of [GPUTextureViewDescriptor](https://gpuweb.github.io#dictdef-gputextureviewdescriptor)
- `[[renderExtent]]`, readonly
- For renderable views, this is the effective [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)
Note: this extent depends on the [baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)

`view`, with

[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)
`desc`, is the subset of the subresources of

`view`.

[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)
`s`satisfies the following:

- The [mipmap level](https://gpuweb.github.io#mipmap-level) of `s` is ≥ `desc`.[baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)`desc`.[baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)`desc`.[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)
- The [array layer](https://gpuweb.github.io#array-layer) of `s` is ≥ `desc`.[baseArrayLayer](https://gpuweb.github.io#dom-gputextureviewdescriptor-basearraylayer)`desc`.[baseArrayLayer](https://gpuweb.github.io#dom-gputextureviewdescriptor-basearraylayer)`desc`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)
- The [aspect](https://gpuweb.github.io#aspect) of `s` is in the [set of aspects](https://gpuweb.github.io#gputextureaspect-set-of-aspects) of `desc`.[aspect](https://gpuweb.github.io#dom-gputextureviewdescriptor-aspect)

Two [GPUTextureView](https://gpuweb.github.io#gputextureview)

#### 6.2.1. Texture View Creation

dictionary GPUTextureViewDescriptor GPUObjectDescriptorBase GPUTextureFormat format GPUTextureViewDimension dimension GPUTextureUsageFlags usage GPUTextureAspect aspect GPUIntegerCoordinate baseMipLevel GPUIntegerCoordinate mipLevelCount GPUIntegerCoordinate baseArrayLayer GPUIntegerCoordinate arrayLayerCount DOMString swizzle 

[GPUTextureViewDescriptor](https://gpuweb.github.io#dictdef-gputextureviewdescriptor)

- `format`,  of type [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- The format of the texture view. Must be either the [format](https://gpuweb.github.io#dom-gputexturedescriptor-format)[viewFormats](https://gpuweb.github.io#dom-gputexturedescriptor-viewformats)
- `dimension`,  of type [GPUTextureViewDimension](https://gpuweb.github.io#enumdef-gputextureviewdimension)
- The dimension to view the texture as.
- `usage`,  of type [GPUTextureUsageFlags](https://gpuweb.github.io#typedefdef-gputextureusageflags), defaulting to `0`
- The allowed [usage(s)](https://gpuweb.github.io#namespacedef-gputextureusage)[usage](https://gpuweb.github.io#dom-gputexture-usage)[usage](https://gpuweb.github.io#dom-gputexture-usage)
Note: If the view’s [format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)[usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)
- `aspect`,  of type [GPUTextureAspect](https://gpuweb.github.io#enumdef-gputextureaspect), defaulting to `"all"`
- Which [aspect(s)](https://gpuweb.github.io#enumdef-gputextureaspect)
- `baseMipLevel`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate), defaulting to `0`
- The first (most detailed) mipmap level accessible to the texture view.
- `mipLevelCount`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate)
- How many mipmap levels, starting with [baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)
- `baseArrayLayer`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate), defaulting to `0`
- The index of the first array layer accessible to the texture view.
- `arrayLayerCount`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate)
- How many array layers, starting with [baseArrayLayer](https://gpuweb.github.io#dom-gputextureviewdescriptor-basearraylayer)
- `swizzle`,  of type [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString), defaulting to `"rgba"`
- A string of length four, with each character mapping to the texture view’s red/green/blue/alpha channels, respectively. When accessed by a shader, the red/green/blue/alpha channels are replaced by the value
corresponding to the component specified in `swizzle[0]`, `swizzle[1]`, `swizzle[2]`, and `swizzle[3]`, respectively:
  - `"r"`: Take its value from the red channel of the texture.
  - `"g"`: Take its value from the green channel of the texture.
  - `"b"`: Take its value from the blue channel of the texture.
  - `"a"`: Take its value from the alpha channel of the texture.
  - `"0"`: Force its value to 0.
  - `"1"`: Force its value to 1.
Requires the ["texture-component-swizzle"](https://gpuweb.github.io#dom-gpufeaturename-texture-component-swizzle)

enum GPUTextureViewDimension "1d" "2d" "2d-array" "cube" "cube-array" "3d" 

- `"1d"`
- The texture is viewed as a 1-dimensional image. Corresponding WGSL types: 
  - `texture_1d`
  - `texture_storage_1d`
- `"2d"`
- The texture is viewed as a single 2-dimensional image. Corresponding WGSL types: 
  - `texture_2d`
  - `texture_storage_2d`
  - `texture_multisampled_2d`
  - `texture_depth_2d`
  - `texture_depth_multisampled_2d`
- `"2d-array"`
- The texture view is viewed as an array of 2-dimensional images. Corresponding WGSL types: 
  - `texture_2d_array`
  - `texture_storage_2d_array`
  - `texture_depth_2d_array`
- `"cube"`
- The texture is viewed as a cubemap. The view has 6 array layers, each corresponding to a face of the cube in the order
`[+X, -X, +Y, -Y, +Z, -Z]` and the following orientations:
Cubemap faces. The +U/+V axes indicate the individual faces' texture coordinates, and thus the [texel copy](https://gpuweb.github.io#texel-copy) memory layout of each face. Note: When viewed from the inside, this results in a left-handed coordinate system where +X is right, +Y is up, and +Z is forward. Sampling is done seamlessly across the faces of the cubemap. Corresponding WGSL types: 
  - `texture_cube`
  - `texture_depth_cube`
- `"cube-array"`
- The texture is viewed as a packed array of `n` cubemaps,
each with 6 array layers behaving like one ["cube"](https://gpuweb.github.io#dom-gputextureviewdimension-cube)`n` array layers in total.
Corresponding WGSL types: 
  - `texture_cube_array`
  - `texture_depth_cube_array`
- `"3d"`
- The texture is viewed as a 3-dimensional image. Corresponding WGSL types: 
  - `texture_3d`
  - `texture_storage_3d`

Each `GPUTextureAspect` value corresponds to a set of [aspects](https://gpuweb.github.io#aspect).
The set of aspects are defined for each value below.

enum GPUTextureAspect "all" "stencil-only" "depth-only" 

- `"all"`
- All available aspects of the texture format will be accessible to the texture view. For
color formats the color aspect will be accessible. For
[combined depth-stencil format](https://gpuweb.github.io#combined-depth-stencil-format)s both the depth and stencil aspects will be accessible. [Depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format)s with a single aspect will only make that aspect accessible.
The [set of aspects](https://gpuweb.github.io#gputextureaspect-set-of-aspects) is \[[color](https://gpuweb.github.io#aspect-color), [depth](https://gpuweb.github.io#aspect-depth), [stencil](https://gpuweb.github.io#aspect-stencil)\].
- `"stencil-only"`
- Only the stencil aspect of a [depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format) format will be accessible to the
texture view.
The [set of aspects](https://gpuweb.github.io#gputextureaspect-set-of-aspects) is \[[stencil](https://gpuweb.github.io#aspect-stencil)\].
- `"depth-only"`
- Only the depth aspect of a [depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format) format will be accessible to the
texture view.
The [set of aspects](https://gpuweb.github.io#gputextureaspect-set-of-aspects) is \[[depth](https://gpuweb.github.io#aspect-depth)\].

- `createView(descriptor)`
- Creates a [GPUTextureView](https://gpuweb.github.io#gputextureview)
NOTE:By default[createView()](https://gpuweb.github.io#dom-gputexture-createview)
[createView()](https://gpuweb.github.io#dom-gputexture-createview)
[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
["2d-array"](https://gpuweb.github.io#dom-gputextureviewdimension-2d-array)
[GPUTextureView](https://gpuweb.github.io#gputextureview)
[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)
For textures created from sources where the layer count is unknown at the
    time of development it is recommended that calls to [createView()](https://gpuweb.github.io#dom-gputexture-createview)[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
**Called on:** [GPUTexture](https://gpuweb.github.io#gputexture)
`this`. **Arguments:**
Arguments for the [GPUTexture.createView(descriptor)](https://gpuweb.github.io#dom-gputexture-createview) method.Parameter Type Nullable Optional Description `descriptor`
[GPUTextureViewDescriptor](https://gpuweb.github.io#dictdef-gputextureviewdescriptor)
✘ ✔ Description of the [GPUTextureView](https://gpuweb.github.io#gputextureview)
**Returns:** `view`, of type [GPUTextureView](https://gpuweb.github.io#gputextureview)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of `descriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  2. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate swizzle string](https://gpuweb.github.io#abstract-opdef-validate-swizzle-string) of `descriptor`.[swizzle](https://gpuweb.github.io#dom-gputextureviewdescriptor-swizzle)
  3. Let `view` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUTextureView](https://gpuweb.github.io#gputextureview)`descriptor`).
  4. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  5. Return `view`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Set `descriptor` to the result of [resolving GPUTextureViewDescriptor defaults](https://gpuweb.github.io#abstract-opdef-resolving-gputextureviewdescriptor-defaults) for `this` with `descriptor`.
  2. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `view` and return.
    - `this` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
    - `descriptor`.[aspect](https://gpuweb.github.io#dom-gputextureviewdescriptor-aspect)`this`.[format](https://gpuweb.github.io#dom-gputexture-format)
    - If the `descriptor`.[aspect](https://gpuweb.github.io#dom-gputextureviewdescriptor-aspect)["all"](https://gpuweb.github.io#dom-gputextureaspect-all)
      - `descriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)`this`.[format](https://gpuweb.github.io#dom-gputexture-format)`this`.[\[\[viewFormats\]\]](https://gpuweb.github.io#dom-gputexture-viewformats-slot)
Otherwise: 
      - `descriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)[resolving GPUTextureAspect](https://gpuweb.github.io#abstract-opdef-resolving-gputextureaspect)( `this`.[format](https://gpuweb.github.io#dom-gputexture-format)`descriptor`.[aspect](https://gpuweb.github.io#dom-gputextureviewdescriptor-aspect)
    - If `descriptor`.[swizzle](https://gpuweb.github.io#dom-gputextureviewdescriptor-swizzle)`"rgba"`, ["texture-component-swizzle"](https://gpuweb.github.io#dom-gpufeaturename-texture-component-swizzle)[enabled for](https://gpuweb.github.io#enabled-for) `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
    - `descriptor`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)`this`.[usage](https://gpuweb.github.io#dom-gputexture-usage)
    - If `this`.[usage](https://gpuweb.github.io#dom-gputexture-usage)[TRANSIENT\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-transient_attachment)
    - If `descriptor`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)
      - `descriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)[renderable format](https://gpuweb.github.io#renderable).
    - If `descriptor`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)
      - `descriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)[§ 26.1.1 Plain color formats](https://gpuweb.github.io#plain-color-formats) table
with [STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)
    - `descriptor`.[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)
    - `descriptor`.[baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)`descriptor`.[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)`this`.[mipLevelCount](https://gpuweb.github.io#dom-gputexture-miplevelcount)
    - `descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)
    - `descriptor`.[baseArrayLayer](https://gpuweb.github.io#dom-gputextureviewdescriptor-basearraylayer)`descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)[array layer count](https://gpuweb.github.io#abstract-opdef-array-layer-count) of `this`.
    - If `this`.[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)`descriptor`.[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)
    - If `descriptor`.[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
      - ["1d"](https://gpuweb.github.io#dom-gputextureviewdimension-1d)
        - `this`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)["1d"](https://gpuweb.github.io#dom-gputexturedimension-1d)
        - `descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)`1`.
      - ["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)
        - `this`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
        - `descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)`1`.
      - ["2d-array"](https://gpuweb.github.io#dom-gputextureviewdimension-2d-array)
      - ["cube"](https://gpuweb.github.io#dom-gputextureviewdimension-cube)
        - `this`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
        - `descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)`6`.
        - `this`.[width](https://gpuweb.github.io#dom-gputexture-width)`this`.[height](https://gpuweb.github.io#dom-gputexture-height)
      - ["cube-array"](https://gpuweb.github.io#dom-gputextureviewdimension-cube-array)
        - `this`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
        - `descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)`6`.
        - `this`.[width](https://gpuweb.github.io#dom-gputexture-width)`this`.[height](https://gpuweb.github.io#dom-gputexture-height)
      - ["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
        - `this`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)
        - `descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)`1`.
  3. Let `view` be a new [GPUTextureView](https://gpuweb.github.io#gputextureview)
  4. Set `view`.[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)`this`.
  5. Set `view`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)`descriptor`.
  6. If `descriptor`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)
    1. Let `renderExtent` be [compute render extent](https://gpuweb.github.io#abstract-opdef-compute-render-extent)(\[`this`.[width](https://gpuweb.github.io#dom-gputexture-width)`this`.[height](https://gpuweb.github.io#dom-gputexture-height)`this`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gputexture-depthorarraylayers)`descriptor`.[baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)
    2. Set `view`.[\[\[renderExtent\]\]](https://gpuweb.github.io#dom-gputextureview-renderextent-slot)`renderExtent`.

[GPUTextureView](https://gpuweb.github.io#gputextureview)
`texture`with a

[GPUTextureViewDescriptor](https://gpuweb.github.io#dictdef-gputextureviewdescriptor)
`descriptor`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. Let `resolved` be a copy of `descriptor`.
2. If `resolved`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)[provided](https://infra.spec.whatwg.org/#map-exists):
3. If `resolved`.[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)[provided](https://infra.spec.whatwg.org/#map-exists):
set `resolved`.[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)`texture`.[mipLevelCount](https://gpuweb.github.io#dom-gputexture-miplevelcount)`resolved`.[baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)
4. If `resolved`.[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)[provided](https://infra.spec.whatwg.org/#map-exists) and `texture`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)
5. If `resolved`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)[provided](https://infra.spec.whatwg.org/#map-exists) and `resolved`.[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
  - ["1d"](https://gpuweb.github.io#dom-gputextureviewdimension-1d)
["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)
["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
  - Set `resolved`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)`1`.
  - ["cube"](https://gpuweb.github.io#dom-gputextureviewdimension-cube)
  - Set `resolved`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)`6`.
  - ["2d-array"](https://gpuweb.github.io#dom-gputextureviewdimension-2d-array)
["cube-array"](https://gpuweb.github.io#dom-gputextureviewdimension-cube-array)
  - Set `resolved`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)[array layer count](https://gpuweb.github.io#abstract-opdef-array-layer-count) of `texture` − `resolved`.[baseArrayLayer](https://gpuweb.github.io#dom-gputextureviewdescriptor-basearraylayer)
6. If `resolved`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)`0`:
set `resolved`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)`texture`.[usage](https://gpuweb.github.io#dom-gputexture-usage)
7. Return `resolved`.

[GPUTexture](https://gpuweb.github.io#gputexture)
`texture`, run the following steps:

1. If `texture`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)
  - ["1d"](https://gpuweb.github.io#dom-gputexturedimension-1d)
["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)
  - Return `1`.
  - ["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
  - Return `texture`.[depthOrArrayLayers](https://gpuweb.github.io#dom-gputexture-depthorarraylayers)

[DOMString](https://webidl.spec.whatwg.org/#idl-DOMString)
`swizzle`, run the following

[content timeline](https://gpuweb.github.io#content-timeline)steps:

1. If `swizzle` does not match the [\[ECMAScript\]](https://gpuweb.github.io#biblio-ecmascript) regexp `^[rgba01]{4}$`:
  1. Throw a [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)

### 6.3. Texture Formats

The name of the format specifies the order of components, bits per component, and data type for the component.

- `r`, `g`, `b`, `a` = red, green, blue, alpha
- `unorm` = unsigned normalized
- `snorm` = signed normalized
- `uint` = unsigned int
- `sint` = signed int
- `float` = floating point

If the format has the `-srgb` suffix, then sRGB conversions from gamma to linear
and vice versa are applied during the reading and writing of color values in the
shader. Compressed texture formats are provided by [features](https://gpuweb.github.io#feature). Their naming
should follow the convention here, with the texture name as a prefix. e.g.
`etc2-rgba8unorm`.

The texel block is a single addressable element of the textures in pixel-based [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

The texel block width and texel block height specifies the dimension of one [texel block](https://gpuweb.github.io#texel-block).

- For pixel-based [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[texel block width](https://gpuweb.github.io#texel-block-width) and [texel block height](https://gpuweb.github.io#texel-block-height) are always 1.
- For block-based compressed [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[texel block width](https://gpuweb.github.io#texel-block-width) is the number of texels in each row of one [texel block](https://gpuweb.github.io#texel-block),
and the [texel block height](https://gpuweb.github.io#texel-block-height) is the number of texel rows in one [texel block](https://gpuweb.github.io#texel-block). See [§ 26.1 Texture Format Capabilities](https://gpuweb.github.io#texture-format-caps) for an exhaustive list
of values for every texture format.

The texel block copy footprint of an [aspect](https://gpuweb.github.io#aspect) of a [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[texel copy](https://gpuweb.github.io#texel-copy), if applicable.

Note:
The texel block memory cost of a [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[texel block](https://gpuweb.github.io#texel-block). It is not fully defined for all formats.
**This value is informative and non-normative.**

enum GPUTextureFormat "r8unorm" "r8snorm" "r8uint" "r8sint" "r16unorm" "r16snorm" "r16uint" "r16sint" "r16float" "rg8unorm" "rg8snorm" "rg8uint" "rg8sint" "r32uint" "r32sint" "r32float" "rg16unorm" "rg16snorm" "rg16uint" "rg16sint" "rg16float" "rgba8unorm" "rgba8unorm-srgb" "rgba8snorm" "rgba8uint" "rgba8sint" "bgra8unorm" "bgra8unorm-srgb" "rgb9e5ufloat" "rgb10a2uint" "rgb10a2unorm" "rg11b10ufloat" "rg32uint" "rg32sint" "rg32float" "rgba16unorm" "rgba16snorm" "rgba16uint" "rgba16sint" "rgba16float" "rgba32uint" "rgba32sint" "rgba32float" "stencil8" "depth16unorm" "depth24plus" "depth24plus-stencil8" "depth32float" "depth32float-stencil8" "bc1-rgba-unorm" "bc1-rgba-unorm-srgb" "bc2-rgba-unorm" "bc2-rgba-unorm-srgb" "bc3-rgba-unorm" "bc3-rgba-unorm-srgb" "bc4-r-unorm" "bc4-r-snorm" "bc5-rg-unorm" "bc5-rg-snorm" "bc6h-rgb-ufloat" "bc6h-rgb-float" "bc7-rgba-unorm" "bc7-rgba-unorm-srgb" "etc2-rgb8unorm" "etc2-rgb8unorm-srgb" "etc2-rgb8a1unorm" "etc2-rgb8a1unorm-srgb" "etc2-rgba8unorm" "etc2-rgba8unorm-srgb" "eac-r11unorm" "eac-r11snorm" "eac-rg11unorm" "eac-rg11snorm" "astc-4x4-unorm" "astc-4x4-unorm-srgb" "astc-5x4-unorm" "astc-5x4-unorm-srgb" "astc-5x5-unorm" "astc-5x5-unorm-srgb" "astc-6x5-unorm" "astc-6x5-unorm-srgb" "astc-6x6-unorm" "astc-6x6-unorm-srgb" "astc-8x5-unorm" "astc-8x5-unorm-srgb" "astc-8x6-unorm" "astc-8x6-unorm-srgb" "astc-8x8-unorm" "astc-8x8-unorm-srgb" "astc-10x5-unorm" "astc-10x5-unorm-srgb" "astc-10x6-unorm" "astc-10x6-unorm-srgb" "astc-10x8-unorm" "astc-10x8-unorm-srgb" "astc-10x10-unorm" "astc-10x10-unorm-srgb" "astc-12x10-unorm" "astc-12x10-unorm-srgb" "astc-12x12-unorm" "astc-12x12-unorm-srgb" 

The depth component of the ["depth24plus"](https://gpuweb.github.io#dom-gputextureformat-depth24plus)["depth24plus-stencil8"](https://gpuweb.github.io#dom-gputextureformat-depth24plus-stencil8)[24-bit depth](https://gpuweb.github.io#24-bit-depth) value or a ["depth32float"](https://gpuweb.github.io#dom-gputextureformat-depth32float)

The [stencil8](https://gpuweb.github.io#dom-gputextureformat-stencil8)

[24-bit depth](https://gpuweb.github.io#24-bit-depth)channels for all values in the representable range (0.0 to 1.0), note that the set of representable values is not an exact superset.

- For [24-bit depth](https://gpuweb.github.io#24-bit-depth), 1 ULP has a constant value of 1 / (2<sup>24</sup> − 1).
- For depth32float, 1 ULP has a variable value no greater than 1 / (2<sup>24</sup>).

A format is renderable if it is either a color renderable format, or a [depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format).
If a format is listed in [§ 26.1.1 Plain color formats](https://gpuweb.github.io#plain-color-formats) with [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[depth-or-stencil formats](https://gpuweb.github.io#depth-or-stencil-format) are renderable.

A [renderable format](https://gpuweb.github.io#renderable) is also blendable
if it can be used with render pipeline blending.
See [§ 26.1 Texture Format Capabilities](https://gpuweb.github.io#texture-format-caps).

A format is filterable if it supports the
[GPUTextureSampleType](https://gpuweb.github.io#enumdef-gputexturesampletype)["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float)["filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-filtering)[GPUSampler](https://gpuweb.github.io#gpusampler)[§ 26.1 Texture Format Capabilities](https://gpuweb.github.io#texture-format-caps).

**Arguments:**

- [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)`format`
- [GPUTextureAspect](https://gpuweb.github.io#enumdef-gputextureaspect)`aspect`

**Returns:** [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)`null`

1. If `aspect` is:
  - ["all"](https://gpuweb.github.io#dom-gputextureaspect-all)
  - Return `format`.
  - ["depth-only"](https://gpuweb.github.io#dom-gputextureaspect-depth-only)
["stencil-only"](https://gpuweb.github.io#dom-gputextureaspect-stencil-only)
  - If `format` is a depth-stencil-format:
Return the [aspect-specific format](https://gpuweb.github.io#aspect-specific-format) of `format` according to [§ 26.1.2 Depth-stencil formats](https://gpuweb.github.io#depth-formats) or `null` if
    the aspect is not present in `format`.
2. Return `null`.

Use of some texture formats require a feature to be enabled on the [GPUDevice](https://gpuweb.github.io#gpudevice)

See [§ 26.1 Texture Format Capabilities](https://gpuweb.github.io#texture-format-caps) for information about which [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
`format`

with logical

[device](https://gpuweb.github.io#device)

`device`, run the following

[content timeline](https://gpuweb.github.io#content-timeline)steps:

1. If `format` requires a feature and `device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)[contain](https://infra.spec.whatwg.org/#list-contain) the feature:
  1. Throw a [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)

### 6.4. `GPUExternalTexture`

A [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)

[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[externalTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-externaltexture)

[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)
*can*be implemented without creating a copy of the imported source, but this depends

[implementation-defined](https://infra.spec.whatwg.org/#implementation-defined)factors. Ownership of the underlying representation may either be exclusive or shared with other owners (such as a video decoder), but this is not visible to the application.

The underlying representation of an external texture is unobservable (except for precise sampling behavior), but typically may include:

- Up to three 2D planes of data (e.g. RGBA, Y+UV, Y+U+V).
- Metadata for converting coordinates before reading from those planes (crop and rotation).
- Metadata for converting values into the specified output color space (matrices, gammas, 3D LUT).

The configuration used internally by an implementation may be inconsistent across time,
    systems, user agents, media sources, or even frames within a single video source.
    In order to account for many possible representations,
    the binding conservatively uses the following, for *each* external texture:

- three sampled texture bindings (for up to 3 planes),
- one sampled texture binding for a 3D LUT,
- one sampler binding to sample the 3D LUT, and
- one uniform buffer binding for metadata.

\[Exposed Window ,Worker ),SecureContext interface GPUExternalTexture GPUExternalTexture includes GPUObjectBase 

[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[immutable properties](https://gpuweb.github.io#immutable-property):

- `[[descriptor]]`, of type [GPUExternalTextureDescriptor](https://gpuweb.github.io#dictdef-gpuexternaltexturedescriptor)
- The descriptor with which the texture was created.

[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[immutable properties](https://gpuweb.github.io#immutable-property):

- `[[expired]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
`false`
- Indicates whether the object has expired (can no longer be used). Note:
Unlike `[[destroyed]]` slots, which are similar, this can change from `true` back to `false`.

#### 6.4.1. Importing External Textures

An external texture is created from an external video object
using [importExternalTexture()](https://gpuweb.github.io#dom-gpudevice-importexternaltexture)

An external texture created from an [HTMLVideoElement](https://html.spec.whatwg.org/multipage/media.html#htmlvideoelement)[\[\[expired\]\]](https://gpuweb.github.io#dom-gpuexternaltexture-expired-slot)`true`.

An external texture created from a [VideoFrame](https://w3c.github.io/webcodecs/#videoframe)[VideoFrame](https://w3c.github.io/webcodecs/#videoframe)[closed](https://www.w3.org/TR/webcodecs/#close-videoframe),
either explicitly by [close()](https://w3c.github.io/webcodecs/#dom-videoframe-close)

Note: As noted in [decode()](https://w3c.github.io/webcodecs/#dom-videodecoder-decode)**should** call
[close()](https://w3c.github.io/webcodecs/#dom-videoframe-close)[VideoFrame](https://w3c.github.io/webcodecs/#videoframe)[VideoFrame](https://w3c.github.io/webcodecs/#videoframe)[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[VideoFrame](https://w3c.github.io/webcodecs/#videoframe)

Once the [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[importExternalTexture()](https://gpuweb.github.io#dom-gpudevice-importexternaltexture)[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)`requestVideoFrameCallback()`).
If the same object is returned again, it will compare equal, and [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)

dictionary GPUExternalTextureDescriptor GPUObjectDescriptorBase required (HTMLVideoElement or VideoFrame source PredefinedColorSpace colorSpace 

[GPUExternalTextureDescriptor](https://gpuweb.github.io#dictdef-gpuexternaltexturedescriptor)

- `source`,  of type `(HTMLVideoElement or VideoFrame)`
- The video source to import the external texture from. Source size is determined as described
by the [external source dimensions](https://gpuweb.github.io#external-source-dimensions) table.
- `colorSpace`,  of type [PredefinedColorSpace](https://html.spec.whatwg.org/multipage/canvas.html#predefinedcolorspace), defaulting to `"srgb"`
- The color space the image contents of [source](https://gpuweb.github.io#dom-gpuexternaltexturedescriptor-source)

- `importExternalTexture(descriptor)`
- Creates a [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.importExternalTexture(descriptor)](https://gpuweb.github.io#dom-gpudevice-importexternaltexture) method.Parameter Type Nullable Optional Description `descriptor`
[GPUExternalTextureDescriptor](https://gpuweb.github.io#dictdef-gpuexternaltexturedescriptor)
✘ ✘ Provides the external image source object (and any creation options). **Returns:** [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `source` be `descriptor`.[source](https://gpuweb.github.io#dom-gpuexternaltexturedescriptor-source)
  2. If the current image contents of `source` are the same as the most recent [importExternalTexture()](https://gpuweb.github.io#dom-gpudevice-importexternaltexture)`descriptor` (ignoring [label](https://gpuweb.github.io#dom-gpuobjectdescriptorbase-label)
    1. Let `previousResult` be the [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)
    2. Set `previousResult`.[\[\[expired\]\]](https://gpuweb.github.io#dom-gpuexternaltexture-expired-slot)`false`,
renewing ownership of the underlying resource.
    3. Let `result` be `previousResult`.
Note:
This allows the application to detect duplicate imports and avoid re-creating
dependent objects (such as [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[colorSpace](https://gpuweb.github.io#dom-gpuexternaltexturedescriptor-colorspace)
Otherwise: 
    1. If `source` [is not origin-clean](https://html.spec.whatwg.org/multipage/canvas.html#the-image-argument-is-not-origin-clean),
throw a [SecurityError](https://webidl.spec.whatwg.org/#securityerror)
    2. Let `usability` be [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [check the usability of the image argument](https://html.spec.whatwg.org/multipage/canvas.html#check-the-usability-of-the-image-argument)(`source`).
    3. If `usability` is not `good`:
      1. [Generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).
      2. Return an [invalidated](https://gpuweb.github.io#abstract-opdef-invalidate) [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)
    4. Let `data` be the result of converting the current image contents of `source` into
the color space `descriptor`.[colorSpace](https://gpuweb.github.io#dom-gpuexternaltexturedescriptor-colorspace)
This [may result](https://gpuweb.github.io#color-space-conversions) in values outside of the range \[0, 1\].
If clamping is desired, it may be performed after sampling.
Note: This is described like a copy, but may be implemented as a reference to read-only underlying data plus appropriate metadata to perform conversion later.
    5. Let `result` be a new [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)`data`.
  3. If `source` is an [HTMLVideoElement](https://html.spec.whatwg.org/multipage/media.html#htmlvideoelement)[queue an automatic expiry task](https://gpuweb.github.io#abstract-opdef-queue-an-automatic-expiry-task) with device `this` and the following steps:
    1. Set `result`.[\[\[expired\]\]](https://gpuweb.github.io#dom-gpuexternaltexture-expired-slot)`true`,
releasing ownership of the underlying resource.
Note:
An [HTMLVideoElement](https://html.spec.whatwg.org/multipage/media.html#htmlvideoelement)`requestVideoFrameCallback` or [requestAnimationFrame()](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#dom-animationframeprovider-requestanimationframe)
  4. If `source` is a [VideoFrame](https://w3c.github.io/webcodecs/#videoframe)`source` is [closed](https://www.w3.org/TR/webcodecs/#close-videoframe), run the following steps:
    1. Set `result`.[\[\[expired\]\]](https://gpuweb.github.io#dom-gpuexternaltexture-expired-slot)`true`.
  5. Set `result`.[label](https://gpuweb.github.io#dom-gpuobjectbase-label)`descriptor`.[label](https://gpuweb.github.io#dom-gpuobjectdescriptorbase-label)
  6. Return `result`.

[Rendering using an video element external texture at the page animation frame rate:](https://gpuweb.github.io#example-bb35d935)

const videoElement= document. createElement( 'video' ); // ... set up videoElement, wait for it to be ready... function frame() { requestAnimationFrame( frame); // Always re-import the video on every animation frame, because the // import is likely to have expired. // The browser may cache and reuse a past frame, and if it does it // may return the same GPUExternalTexture object again. // In this case, old bind groups are still valid. const externalTexture= gpuDevice. importExternalTexture({ source: videoElement}); // ... render using externalTexture... } requestAnimationFrame( frame); 

[Rendering using an video element external texture at the video’s frame rate, if](https://gpuweb.github.io#example-8be61f91)

`requestVideoFrameCallback` is available:
    
const videoElement= document. createElement( 'video' ); // ... set up videoElement... function frame() { videoElement. requestVideoFrameCallback( frame); // Always re-import, because we know the video frame has advanced const externalTexture= gpuDevice. importExternalTexture({ source: videoElement}); // ... render using externalTexture... } videoElement. requestVideoFrameCallback( frame); 

### 6.5. Sampling External Texture Bindings

The [externalTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-externaltexture)[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)

Note:
When a [GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)[externalTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-externaltexture)[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)

External textures are represented in WGSL with `texture_external` and may be read using
`textureLoad` and `textureSampleBaseClampToEdge`.

The `sampler` provided to `textureSampleBaseClampToEdge` is used to sample the underlying textures.

When the [binding resource type](https://gpuweb.github.io#binding-resource-type) is a [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[colorSpace](https://gpuweb.github.io#dom-gpuexternaltexturedescriptor-colorspace)

Note: If the internal representation is an RGBA plane, sampling behaves as on a regular 2D texture. If there are several underlying planes (e.g. Y+UV), the sampler is used to sample each underlying texture separately, prior to conversion from YUV to the specified color space.

## 7. Samplers

### 7.1. `GPUSampler`

A [GPUSampler](https://gpuweb.github.io#gpusampler)

[GPUSampler](https://gpuweb.github.io#gpusampler)[createSampler()](https://gpuweb.github.io#dom-gpudevice-createsampler)

\[Exposed Window ,Worker ),SecureContext interface GPUSampler GPUSampler includes GPUObjectBase 

[GPUSampler](https://gpuweb.github.io#gpusampler)[immutable properties](https://gpuweb.github.io#immutable-property):

- `[[descriptor]]`, of type [GPUSamplerDescriptor](https://gpuweb.github.io#dictdef-gpusamplerdescriptor)
- The [GPUSamplerDescriptor](https://gpuweb.github.io#dictdef-gpusamplerdescriptor)[GPUSampler](https://gpuweb.github.io#gpusampler)
- `[[isComparison]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- Whether the [GPUSampler](https://gpuweb.github.io#gpusampler)
- `[[isFiltering]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- Whether the [GPUSampler](https://gpuweb.github.io#gpusampler)

#### 7.1.1. [GPUSamplerDescriptor](https://gpuweb.github.io#dictdef-gpusamplerdescriptor)

A [GPUSamplerDescriptor](https://gpuweb.github.io#dictdef-gpusamplerdescriptor)[GPUSampler](https://gpuweb.github.io#gpusampler)

dictionary GPUSamplerDescriptor GPUObjectDescriptorBase GPUAddressMode addressModeU GPUAddressMode addressModeV GPUAddressMode addressModeW GPUFilterMode magFilter GPUFilterMode minFilter GPUMipmapFilterMode mipmapFilter float lodMinClamp float lodMaxClamp GPUCompareFunction compare Clamp unsigned short maxAnisotropy 

- `addressModeU`,  of type [GPUAddressMode](https://gpuweb.github.io#enumdef-gpuaddressmode), defaulting to `"clamp-to-edge"` `addressModeV`,  of type [GPUAddressMode](https://gpuweb.github.io#enumdef-gpuaddressmode), defaulting to `"clamp-to-edge"` `addressModeW`,  of type [GPUAddressMode](https://gpuweb.github.io#enumdef-gpuaddressmode), defaulting to `"clamp-to-edge"`
- Specifies the [address modes](https://gpuweb.github.io#enumdef-gpuaddressmode)
- `magFilter`,  of type [GPUFilterMode](https://gpuweb.github.io#enumdef-gpufiltermode), defaulting to `"nearest"`
- Specifies the sampling behavior when the sampled area is smaller than or equal to one texel.
- `minFilter`,  of type [GPUFilterMode](https://gpuweb.github.io#enumdef-gpufiltermode), defaulting to `"nearest"`
- Specifies the sampling behavior when the sampled area is larger than one texel.
- `mipmapFilter`,  of type [GPUMipmapFilterMode](https://gpuweb.github.io#enumdef-gpumipmapfiltermode), defaulting to `"nearest"`
- Specifies behavior for sampling between mipmap levels.
- `lodMinClamp`,  of type [float](https://webidl.spec.whatwg.org/#idl-float), defaulting to `0` `lodMaxClamp`,  of type [float](https://webidl.spec.whatwg.org/#idl-float), defaulting to `32`
- Specifies the minimum and maximum [levels of detail](https://gpuweb.github.io#levels-of-detail), respectively, used internally when
sampling a texture.
- `compare`,  of type [GPUCompareFunction](https://gpuweb.github.io#enumdef-gpucomparefunction)
- When provided the sampler will be a comparison sampler with the specified
[GPUCompareFunction](https://gpuweb.github.io#enumdef-gpucomparefunction)
Note: Comparison samplers may use filtering, but the sampling results will be implementation-dependent and may differ from the normal filtering rules.
- `maxAnisotropy`,  of type [unsigned short](https://webidl.spec.whatwg.org/#idl-unsigned-short), defaulting to `1`
- Specifies the maximum anisotropy value clamp used by the sampler. Anisotropic filtering is
enabled when [maxAnisotropy](https://gpuweb.github.io#dom-gpusamplerdescriptor-maxanisotropy)
Anisotropic filtering improves the image quality of textures sampled at oblique viewing
angles. Higher [maxAnisotropy](https://gpuweb.github.io#dom-gpusamplerdescriptor-maxanisotropy)
NOTE:Most implementations support[maxAnisotropy](https://gpuweb.github.io#dom-gpusamplerdescriptor-maxanisotropy)
[maxAnisotropy](https://gpuweb.github.io#dom-gpusamplerdescriptor-maxanisotropy)
The precise filtering behavior is implementation-dependent.

Level of detail (LOD) describes which mip level(s) are selected when sampling a
texture. It may be specified explicitly through shader methods like [textureSampleLevel](https://gpuweb.github.io/gpuweb/wgsl/#texturesamplelevel) or implicitly determined from
the [texture coordinate](https://gpuweb.github.io#texture-coordinates) derivatives.

Note: See [Scale Factor Operation, LOD Operation and Image Level Selection](https://registry.khronos.org/vulkan/specs/1.3/html/vkspec.html#textures-lod-and-scale-factor) in
the [Vulkan 1.3](https://registry.khronos.org/vulkan/specs/1.3/html/vkspec.html) spec for an example of how implicit LODs may be calculated.

[GPUAddressMode](https://gpuweb.github.io#enumdef-gpuaddressmode)

enum GPUAddressMode "clamp-to-edge" "repeat" "mirror-repeat" 

- `"clamp-to-edge"`
- Texture coordinates are clamped between 0.0 and 1.0, inclusive.
- `"repeat"`
- Texture coordinates wrap to the other side of the texture.
- `"mirror-repeat"`
- Texture coordinates wrap to the other side of the texture, but the texture is flipped when the integer part of the coordinate is odd.

[GPUFilterMode](https://gpuweb.github.io#enumdef-gpufiltermode)[GPUMipmapFilterMode](https://gpuweb.github.io#enumdef-gpumipmapfiltermode)

Note: See [Texel Filtering](https://registry.khronos.org/vulkan/specs/1.3/html/vkspec.html#textures-texel-filtering) in the [Vulkan 1.3](https://registry.khronos.org/vulkan/specs/1.3/html/vkspec.html) spec for an example of how
samplers may determine which texels are sampled from for the various filtering modes.

enum GPUFilterMode "nearest" "linear" enum GPUMipmapFilterMode "nearest" "linear" 

- `"nearest"`
- Return the value of the texel nearest to the texture coordinates.
- `"linear"`
- Select two texels in each dimension and return a linear interpolation between their values.

[GPUCompareFunction](https://gpuweb.github.io#enumdef-gpucomparefunction)`depth_ref` is compared to the fetched texel value, and the result of this
comparison test is generated (`1.0f` for pass, or `0.0f` for fail).

After comparison, if texture filtering is enabled, the filtering step occurs, so that comparison
results are mixed together resulting in values in the range `[0, 1]`. Filtering **should** behave
as usual, however it **may** be computed with lower precision or not mix results at all.

enum GPUCompareFunction "never" "less" "equal" "less-equal" "greater" "not-equal" "greater-equal" "always" 

- `"never"`
- Comparison tests never pass.
- `"less"`
- A provided value passes the comparison test if it is less than the sampled value.
- `"equal"`
- A provided value passes the comparison test if it is equal to the sampled value.
- `"less-equal"`
- A provided value passes the comparison test if it is less than or equal to the sampled value.
- `"greater"`
- A provided value passes the comparison test if it is greater than the sampled value.
- `"not-equal"`
- A provided value passes the comparison test if it is not equal to the sampled value.
- `"greater-equal"`
- A provided value passes the comparison test if it is greater than or equal to the sampled value.
- `"always"`
- Comparison tests always pass.

#### 7.1.2. Sampler Creation

- `createSampler(descriptor)`
- Creates a [GPUSampler](https://gpuweb.github.io#gpusampler)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
**Arguments:**
Arguments for the [GPUDevice.createSampler(descriptor)](https://gpuweb.github.io#dom-gpudevice-createsampler) method.Parameter Type Nullable Optional Description `descriptor`
[GPUSamplerDescriptor](https://gpuweb.github.io#dictdef-gpusamplerdescriptor)
✘ ✔ Description of the [GPUSampler](https://gpuweb.github.io#gpusampler)
**Returns:** [GPUSampler](https://gpuweb.github.io#gpusampler)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `s` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUSampler](https://gpuweb.github.io#gpusampler)`descriptor`).
  2. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  3. Return `s`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `s` and return.
    - `this` must not be [lost](https://gpuweb.github.io#abstract-opdef-invalid).
    - `descriptor`.[lodMinClamp](https://gpuweb.github.io#dom-gpusamplerdescriptor-lodminclamp)
    - `descriptor`.[lodMaxClamp](https://gpuweb.github.io#dom-gpusamplerdescriptor-lodmaxclamp)`descriptor`.[lodMinClamp](https://gpuweb.github.io#dom-gpusamplerdescriptor-lodminclamp)
    - `descriptor`.[maxAnisotropy](https://gpuweb.github.io#dom-gpusamplerdescriptor-maxanisotropy)
Note: Most implementations support [maxAnisotropy](https://gpuweb.github.io#dom-gpusamplerdescriptor-maxanisotropy)[maxAnisotropy](https://gpuweb.github.io#dom-gpusamplerdescriptor-maxanisotropy)
    - If `descriptor`.[maxAnisotropy](https://gpuweb.github.io#dom-gpusamplerdescriptor-maxanisotropy)
      - `descriptor`.[magFilter](https://gpuweb.github.io#dom-gpusamplerdescriptor-magfilter)`descriptor`.[minFilter](https://gpuweb.github.io#dom-gpusamplerdescriptor-minfilter)`descriptor`.[mipmapFilter](https://gpuweb.github.io#dom-gpusamplerdescriptor-mipmapfilter)["linear"](https://gpuweb.github.io#dom-gpumipmapfiltermode-linear)
  2. Set `s`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpusampler-descriptor-slot)`descriptor`.
  3. Set `s`.[\[\[isComparison\]\]](https://gpuweb.github.io#dom-gpusampler-iscomparison-slot)`false` if the [compare](https://gpuweb.github.io#dom-gpusamplerdescriptor-compare)`s`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpusampler-descriptor-slot)`null` or undefined. Otherwise, set it to `true`.
  4. Set `s`.[\[\[isFiltering\]\]](https://gpuweb.github.io#dom-gpusampler-isfiltering-slot)`false` if none of [minFilter](https://gpuweb.github.io#dom-gpusamplerdescriptor-minfilter)[magFilter](https://gpuweb.github.io#dom-gpusamplerdescriptor-magfilter)[mipmapFilter](https://gpuweb.github.io#dom-gpusamplerdescriptor-mipmapfilter)["linear"](https://gpuweb.github.io#dom-gpufiltermode-linear)`true`.

[Creating a](https://gpuweb.github.io#example-118a2819)

[GPUSampler](https://gpuweb.github.io#gpusampler)
const sampler= gpuDevice. createSampler({ addressModeU: 'repeat' , addressModeV: 'repeat' , magFilter: 'linear' , minFilter: 'linear' , mipmapFilter: 'linear' , }); 

## 8. Resource Binding

### 8.1. `GPUBindGroupLayout`

A [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)

\[Exposed Window ,Worker ),SecureContext interface GPUBindGroupLayout GPUBindGroupLayout includes GPUObjectBase 

[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[immutable properties](https://gpuweb.github.io#immutable-property):

- `[[descriptor]]`, of type [GPUBindGroupLayoutDescriptor](https://gpuweb.github.io#dictdef-gpubindgrouplayoutdescriptor)

#### 8.1.1. Bind Group Layout Creation

A [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUDevice.createBindGroupLayout()](https://gpuweb.github.io#dom-gpudevice-createbindgrouplayout)

dictionary GPUBindGroupLayoutDescriptor GPUObjectDescriptorBase required sequence GPUBindGroupLayoutEntry entries 

[GPUBindGroupLayoutDescriptor](https://gpuweb.github.io#dictdef-gpubindgrouplayoutdescriptor)

- `entries`,  of type sequence\<[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)>
- A list of entries describing the shader resource bindings for a bind group.

A [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)

dictionary GPUBindGroupLayoutEntry required GPUIndex32 binding required GPUShaderStageFlags visibility GPUBufferBindingLayout buffer GPUSamplerBindingLayout sampler GPUTextureBindingLayout texture GPUStorageTextureBindingLayout storageTexture GPUExternalTextureBindingLayout externalTexture 

[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)

- `binding`,  of type [GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)
- A unique identifier for a resource binding within the [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUBindGroupEntry.binding](https://gpuweb.github.io#dom-gpubindgroupentry-binding)[@binding](https://gpuweb.github.io/gpuweb/wgsl/#attribute-binding) attribute in the [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
- `visibility`,  of type [GPUShaderStageFlags](https://gpuweb.github.io#typedefdef-gpushaderstageflags)
- A bitset of the members of [GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage)[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)
- `buffer`,  of type [GPUBufferBindingLayout](https://gpuweb.github.io#dictdef-gpubufferbindinglayout) `sampler`,  of type [GPUSamplerBindingLayout](https://gpuweb.github.io#dictdef-gpusamplerbindinglayout) `texture`,  of type [GPUTextureBindingLayout](https://gpuweb.github.io#dictdef-gputexturebindinglayout) `storageTexture`,  of type [GPUStorageTextureBindingLayout](https://gpuweb.github.io#dictdef-gpustoragetexturebindinglayout) `externalTexture`,  of type [GPUExternalTextureBindingLayout](https://gpuweb.github.io#dictdef-gpuexternaltexturebindinglayout)
- Exactly one of these members must be set, indicating the binding type. The contents of the member specify options specific to that type. The corresponding resource in [createBindGroup()](https://gpuweb.github.io#dom-gpudevice-createbindgroup)[binding resource type](https://gpuweb.github.io#binding-resource-type) for this binding.

typedef \[EnforceRange unsigned long GPUShaderStageFlags Exposed Window ,Worker ),SecureContext namespace GPUShaderStage const GPUFlagsConstant VERTEX const GPUFlagsConstant FRAGMENT const GPUFlagsConstant COMPUTE 

[GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage)[GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)

- `VERTEX`
- The bind group entry will be accessible to vertex shaders.
- `FRAGMENT`
- The bind group entry will be accessible to fragment shaders.
- `COMPUTE`
- The bind group entry will be accessible to compute shaders.

The [binding member](https://gpuweb.github.io#binding-member) of a [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[externalTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-externaltexture)[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[GPUBindingResource](https://gpuweb.github.io#typedefdef-gpubindingresource)[binding type](https://gpuweb.github.io#binding-type) has an associated [internal usage](https://gpuweb.github.io#internal-usage), given by this table:

| Binding member | Resource type | Binding type | Binding usage | 
|---|---|---|---|
| [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer) | [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding) (or [GPUBuffer](https://gpuweb.github.io#gpubuffer) as [shorthand](https://gpuweb.github.io#abstract-opdef-get-as-buffer-binding)) | ["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform) | [constant](https://gpuweb.github.io#internal-usage-constant) | 
|  |  | ["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage) | [storage](https://gpuweb.github.io#internal-usage-storage) | 
|  |  | ["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage) | [storage-read](https://gpuweb.github.io#internal-usage-storage-read) | 
| [sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler) | [GPUSampler](https://gpuweb.github.io#gpusampler) | ["filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-filtering) | [constant](https://gpuweb.github.io#internal-usage-constant) | 
|  |  | ["non-filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-non-filtering) |  | 
|  |  | ["comparison"](https://gpuweb.github.io#dom-gpusamplerbindingtype-comparison) |  | 
| [texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture) | [GPUTextureView](https://gpuweb.github.io#gputextureview) (or [GPUTexture](https://gpuweb.github.io#gputexture) as [shorthand](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)) | ["float"](https://gpuweb.github.io#dom-gputexturesampletype-float) | [constant](https://gpuweb.github.io#internal-usage-constant) | 
|  |  | ["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float) |  | 
|  |  | ["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth) |  | 
|  |  | ["sint"](https://gpuweb.github.io#dom-gputexturesampletype-sint) |  | 
|  |  | ["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint) |  | 
| [storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture) | [GPUTextureView](https://gpuweb.github.io#gputextureview) (or [GPUTexture](https://gpuweb.github.io#gputexture) as [shorthand](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)) | ["write-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-write-only) | [storage](https://gpuweb.github.io#internal-usage-storage) | 
|  |  | ["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write) |  | 
|  |  | ["read-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-only) | [storage-read](https://gpuweb.github.io#internal-usage-storage-read) | 
| [externalTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-externaltexture) | [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture) or [GPUTextureView](https://gpuweb.github.io#gputextureview) (or [GPUTexture](https://gpuweb.github.io#gputexture) as [shorthand](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)) |  | [constant](https://gpuweb.github.io#internal-usage-constant) | 

[list](https://infra.spec.whatwg.org/#list)of

[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)
`entries`exceeds the binding slot limits of

[supported limits](https://gpuweb.github.io#supported-limits)

`limits`if the number of slots used toward a limit exceeds the supported value in

`limits`. Each entry may use multiple slots toward multiple limits.

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. For each `entry` in `entries`, if:
  - `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)
["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform)
`entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)
`true`
  - Consider 1 [maxDynamicUniformBuffersPerPipelineLayout](https://gpuweb.github.io#dom-supported-limits-maxdynamicuniformbuffersperpipelinelayout)
  - `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)
["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)
`entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)
`true`
  - Consider 1 [maxDynamicStorageBuffersPerPipelineLayout](https://gpuweb.github.io#dom-supported-limits-maxdynamicstoragebuffersperpipelinelayout)
2. For each shader stage `stage` in
« [VERTEX](https://gpuweb.github.io#dom-gpushaderstage-vertex)[FRAGMENT](https://gpuweb.github.io#dom-gpushaderstage-fragment)[COMPUTE](https://gpuweb.github.io#dom-gpushaderstage-compute)
  1. For each `entry` in `entries` for which `entry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)`stage`, if:
    - `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)
["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform)
    - Consider 1 [maxUniformBuffersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxuniformbufferspershaderstage)
    - `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)
["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)
["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage)
    - If `stage` is:
      - [VERTEX](https://gpuweb.github.io#dom-gpushaderstage-vertex)
      - Consider 1 [maxStorageBuffersInVertexStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebuffersinvertexstage)
      - [FRAGMENT](https://gpuweb.github.io#dom-gpushaderstage-fragment)
      - Consider 1 [maxStorageBuffersInFragmentStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebuffersinfragmentstage)
      - [COMPUTE](https://gpuweb.github.io#dom-gpushaderstage-compute)
      - Consider 1 [maxStorageBuffersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragebufferspershaderstage)
    - `entry`.[sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)
[provided](https://infra.spec.whatwg.org/#map-exists)
    - Consider 1 [maxSamplersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxsamplerspershaderstage)
    - `entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)
[provided](https://infra.spec.whatwg.org/#map-exists)
    - Consider 1 [maxSampledTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxsampledtexturespershaderstage)
    - `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)
[provided](https://infra.spec.whatwg.org/#map-exists)
    - If `stage` is:
      - [VERTEX](https://gpuweb.github.io#dom-gpushaderstage-vertex)
      - Consider 1 [maxStorageTexturesInVertexStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturesinvertexstage)
      - [FRAGMENT](https://gpuweb.github.io#dom-gpushaderstage-fragment)
      - Consider 1 [maxStorageTexturesInFragmentStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturesinfragmentstage)
      - [COMPUTE](https://gpuweb.github.io#dom-gpushaderstage-compute)
      - Consider 1 [maxStorageTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxstoragetexturespershaderstage)
    - `entry`.[externalTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-externaltexture)
[provided](https://infra.spec.whatwg.org/#map-exists)
    - Consider
4 [maxSampledTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxsampledtexturespershaderstage)[maxSamplersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxsamplerspershaderstage)[maxUniformBuffersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxuniformbufferspershaderstage)
Note: See [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)

enum GPUBufferBindingType "uniform" "storage" "read-only-storage" dictionary GPUBufferBindingLayout GPUBufferBindingType type boolean hasDynamicOffset false ;GPUSize64 minBindingSize 

[GPUBufferBindingLayout](https://gpuweb.github.io#dictdef-gpubufferbindinglayout)

- `type`,  of type [GPUBufferBindingType](https://gpuweb.github.io#enumdef-gpubufferbindingtype), defaulting to `"uniform"`
- Indicates the type required for buffers bound to this binding.
- `hasDynamicOffset`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- Indicates whether this binding requires a dynamic offset.
- `minBindingSize`,  of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64), defaulting to `0`
- Indicates the minimum [size](https://gpuweb.github.io#dom-gpubufferbinding-size)
Bindings are always validated against this size in [createBindGroup()](https://gpuweb.github.io#dom-gpudevice-createbindgroup)
If this *is not* `0`, pipeline creation additionally [validates](https://gpuweb.github.io#abstract-opdef-validating-shader-binding) that this value ≥ the [minimum buffer binding size](https://gpuweb.github.io#minimum-buffer-binding-size) of the variable.
If this *is* `0`, it is ignored by pipeline creation, and instead draw/dispatch commands [validate](https://gpuweb.github.io#abstract-opdef-validate-encoder-bind-groups) that each binding in the [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[minimum buffer binding size](https://gpuweb.github.io#minimum-buffer-binding-size) of the variable.
Note:
Similar execution-time validation is theoretically possible for other
binding-related fields specified for early validation, like
[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)[format](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-format)[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)

enum GPUSamplerBindingType "filtering" "non-filtering" "comparison" dictionary GPUSamplerBindingLayout GPUSamplerBindingType type 

[GPUSamplerBindingLayout](https://gpuweb.github.io#dictdef-gpusamplerbindinglayout)

- `type`,  of type [GPUSamplerBindingType](https://gpuweb.github.io#enumdef-gpusamplerbindingtype), defaulting to `"filtering"`
- Indicates the required type of a sampler bound to this binding.

enum GPUTextureSampleType "float" "unfilterable-float" "depth" "sint" "uint" dictionary GPUTextureBindingLayout GPUTextureSampleType sampleType GPUTextureViewDimension viewDimension boolean multisampled false ; };

[GPUTextureBindingLayout](https://gpuweb.github.io#dictdef-gputexturebindinglayout)

- `sampleType`,  of type [GPUTextureSampleType](https://gpuweb.github.io#enumdef-gputexturesampletype), defaulting to `"float"`
- Indicates the type required for texture views bound to this binding.
- `viewDimension`,  of type [GPUTextureViewDimension](https://gpuweb.github.io#enumdef-gputextureviewdimension), defaulting to `"2d"`
- Indicates the required [dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
- `multisampled`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- Indicates whether or not texture views bound to this binding must be multisampled.

enum GPUStorageTextureAccess "write-only" "read-only" "read-write" dictionary GPUStorageTextureBindingLayout GPUStorageTextureAccess access required GPUTextureFormat format GPUTextureViewDimension viewDimension 

[GPUStorageTextureBindingLayout](https://gpuweb.github.io#dictdef-gpustoragetexturebindinglayout)

- `access`,  of type [GPUStorageTextureAccess](https://gpuweb.github.io#enumdef-gpustoragetextureaccess), defaulting to `"write-only"`
- The access mode for this binding, indicating readability and writability.
- `format`,  of type [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- The required [format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)
- `viewDimension`,  of type [GPUTextureViewDimension](https://gpuweb.github.io#enumdef-gputextureviewdimension), defaulting to `"2d"`
- Indicates the required [dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)

dictionary GPUExternalTextureBindingLayout 

A [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[entryMap]]`, of type [ordered map](https://infra.spec.whatwg.org/#ordered-map)\<[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)
- The map of binding indices pointing to the [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
- `[[dynamicOffsetCount]]`, of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- The number of buffer bindings with dynamic offsets in this [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
- `[[exclusivePipeline]]`, of type [GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)
- The pipeline that created this [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[default pipeline layout](https://gpuweb.github.io#default-pipeline-layout). If not `null`, [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)

- `createBindGroupLayout(descriptor)`
- Creates a [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createBindGroupLayout(descriptor)](https://gpuweb.github.io#dom-gpudevice-createbindgrouplayout) method.Parameter Type Nullable Optional Description `descriptor`
[GPUBindGroupLayoutDescriptor](https://gpuweb.github.io#dictdef-gpubindgrouplayoutdescriptor)
✘ ✘ Description of the [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
**Returns:** [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. For each [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`entry` in `descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)
    1. If `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[provided](https://infra.spec.whatwg.org/#map-exists):
      1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) for `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[format](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-format)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  2. Let `layout` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)`descriptor`).
  3. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  4. Return `layout`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `layout` and return.
    - `this` must not be [lost](https://gpuweb.github.io#abstract-opdef-invalid).
    - Let `limits` be `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)
    - The [binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)`descriptor` is unique.
    - The [binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)`descriptor` must be \< `limits`.[maxBindingsPerBindGroup](https://gpuweb.github.io#dom-supported-limits-maxbindingsperbindgroup)
    - `descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)[exceed the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits) of `limits`.
    - For each [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`entry` in `descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)
      - Exactly one of
`entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)`entry`.[sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)`entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)`entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)`entry`.[externalTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-externaltexture)[provided](https://infra.spec.whatwg.org/#map-exists).
      - `entry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)[GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage)
      - If `entry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)[VERTEX](https://gpuweb.github.io#dom-gpushaderstage-vertex)
        - If `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[provided](https://infra.spec.whatwg.org/#map-exists), `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform)["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage)
        - If `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[provided](https://infra.spec.whatwg.org/#map-exists), `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)["read-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-only)
      - If `entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[multisampled](https://gpuweb.github.io#dom-gputexturebindinglayout-multisampled)`true`:
        - `entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[viewDimension](https://gpuweb.github.io#dom-gputexturebindinglayout-viewdimension)["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)
        - `entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)
      - If `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[provided](https://infra.spec.whatwg.org/#map-exists):
        - `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[viewDimension](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-viewdimension)["cube"](https://gpuweb.github.io#dom-gputextureviewdimension-cube)["cube-array"](https://gpuweb.github.io#dom-gputextureviewdimension-cube-array)
        - `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[format](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-format)`entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)[§ 26.1.1 Plain color formats](https://gpuweb.github.io#plain-color-formats) table.
  2. Set `layout`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-descriptor-slot)`descriptor`.
  3. Set `layout`.[\[\[dynamicOffsetCount\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-dynamicoffsetcount-slot)`descriptor` where [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[provided](https://infra.spec.whatwg.org/#map-exists) and [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)`true`.
  4. Set `layout`.[\[\[exclusivePipeline\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-exclusivepipeline-slot)`null`.
  5. For each [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`entry` in `descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)
    1. Insert `entry` into `layout`.[\[\[entryMap\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-entrymap-slot)`entry`.[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)

#### 8.1.2. Compatibility

[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
`a`and

`b`are considered group-equivalent if and only if all of the following conditions are satisfied:

- `a`.[\[\[exclusivePipeline\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-exclusivepipeline-slot)`b`.[\[\[exclusivePipeline\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-exclusivepipeline-slot)
- for any binding number `binding`, one of the following conditions is satisfied:
  - it’s missing from both `a`.[\[\[entryMap\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-entrymap-slot)`b`.[\[\[entryMap\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-entrymap-slot)
  - `a`.[\[\[entryMap\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-entrymap-slot)`binding`\] == `b`.[\[\[entryMap\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-entrymap-slot)`binding`\]

If bind groups layouts are [group-equivalent](https://gpuweb.github.io#group-equivalent) they can be interchangeably used in all contents.

### 8.2. `GPUBindGroup`

A [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)

\[Exposed Window ,Worker ),SecureContext interface GPUBindGroup GPUBindGroup includes GPUObjectBase 

[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[layout]]`, of type [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
- The [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
- `[[entries]]`, of type [sequence](https://webidl.spec.whatwg.org/#idl-sequence)\<[GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)
- The set of [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
- `[[usedResources]]`, of type [usage scope](https://gpuweb.github.io#usage-scope), readonly
- The set of buffer and texture [subresource](https://gpuweb.github.io#subresource)s used by this bind group,
associated with lists of the [internal usage](https://gpuweb.github.io#internal-usage) flags.

[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
`bindGroup`, given

[list](https://infra.spec.whatwg.org/#list)\<GPUBufferDynamicOffset>

`dynamicOffsets`, are computed as follows:

1. Let `result` be a new [set](https://infra.spec.whatwg.org/#ordered-set)\<([GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)
2. Let `dynamicOffsetIndex` be 0.
3. For each [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)`bindGroupEntry` in `bindGroup`.[\[\[entries\]\]](https://gpuweb.github.io#dom-gpubindgroup-entries-slot)`bindGroupEntry`.[binding](https://gpuweb.github.io#dom-gpubindgroupentry-binding)
  1. Let `bindGroupLayoutEntry` be `bindGroup`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpubindgroup-layout-slot)[\[\[entryMap\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-entrymap-slot)`bindGroupEntry`.[binding](https://gpuweb.github.io#dom-gpubindgroupentry-binding)
  2. If `bindGroupLayoutEntry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[provided](https://infra.spec.whatwg.org/#map-exists), **continue**.
  3. Let `bound` be [get as buffer binding](https://gpuweb.github.io#abstract-opdef-get-as-buffer-binding)(`bindGroupEntry`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)
  4. If `bindGroupLayoutEntry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)
    1. Increment `bound`.[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)`dynamicOffsets`\[`dynamicOffsetIndex`\].
    2. Increment `dynamicOffsetIndex` by 1.
  5. [Append](https://infra.spec.whatwg.org/#set-append) (`bindGroupLayoutEntry`, `bound`) to `result`.
4. Return `result`.

#### 8.2.1. Bind Group Creation

A [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[GPUDevice.createBindGroup()](https://gpuweb.github.io#dom-gpudevice-createbindgroup)

dictionary GPUBindGroupDescriptor GPUObjectDescriptorBase required GPUBindGroupLayout layout required sequence GPUBindGroupEntry entries 

[GPUBindGroupDescriptor](https://gpuweb.github.io#dictdef-gpubindgroupdescriptor)

- `layout`,  of type [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
- The [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
- `entries`,  of type sequence\<[GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)>
- A list of entries describing the resources to expose to the shader for each binding
described by the [layout](https://gpuweb.github.io#dom-gpubindgroupdescriptor-layout)

typedef (GPUSampler or GPUTexture or GPUTextureView or GPUBuffer or GPUBufferBinding or GPUExternalTexture GPUBindingResource dictionary GPUBindGroupEntry required GPUIndex32 binding required GPUBindingResource resource 

A [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)

- `binding`,  of type [GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)
- A unique identifier for a resource binding within the [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[GPUBindGroupLayoutEntry.binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)[@binding](https://gpuweb.github.io/gpuweb/wgsl/#attribute-binding) attribute in the [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
- `resource`,  of type [GPUBindingResource](https://gpuweb.github.io#typedefdef-gpubindingresource)
- The resource to bind, which may be a [GPUSampler](https://gpuweb.github.io#gpusampler)[GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)[GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)

[GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[prevalidatedSize]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- Whether or not this binding entry had its buffer size validated at time of creation.

dictionary GPUBufferBinding required GPUBuffer buffer GPUSize64 offset GPUSize64 size 

A [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)

- `buffer`,  of type [GPUBuffer](https://gpuweb.github.io#gpubuffer)
- The [GPUBuffer](https://gpuweb.github.io#gpubuffer)
- `offset`,  of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64), defaulting to `0`
- The offset, in bytes, from the beginning of [buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)
- `size`,  of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The size, in bytes, of the buffer binding.
If not [provided](https://infra.spec.whatwg.org/#map-exists), specifies the range starting at [offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)[buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)

- `createBindGroup(descriptor)`
- Creates a [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createBindGroup(descriptor)](https://gpuweb.github.io#dom-gpudevice-createbindgroup) method.Parameter Type Nullable Optional Description `descriptor`
[GPUBindGroupDescriptor](https://gpuweb.github.io#dictdef-gpubindgroupdescriptor)
✘ ✘ Description of the [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
**Returns:** [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `bindGroup` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)`descriptor`).
  2. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  3. Return `bindGroup`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Let `limits` be `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)
  2. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `bindGroup` and return.
    - `descriptor`.[layout](https://gpuweb.github.io#dom-gpubindgroupdescriptor-layout)[valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - The number of [entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)`descriptor`.[layout](https://gpuweb.github.io#dom-gpubindgroupdescriptor-layout)`descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgroupdescriptor-entries)
For each [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)`bindingDescriptor` in `descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgroupdescriptor-entries)
    - Let `resource` be `bindingDescriptor`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)
    - There is exactly one [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`layoutBinding` in `descriptor`.[layout](https://gpuweb.github.io#dom-gpubindgroupdescriptor-layout)[entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)`layoutBinding`.[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)`bindingDescriptor`.[binding](https://gpuweb.github.io#dom-gpubindgroupentry-binding)
    - If the defined [binding member](https://gpuweb.github.io#binding-member) for `layoutBinding` is:
      - [sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)
        - `resource` is a [GPUSampler](https://gpuweb.github.io#gpusampler)
        - `resource` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
        - If `layoutBinding`.[sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)[type](https://gpuweb.github.io#dom-gpusamplerbindinglayout-type)
          - ["filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-filtering)
          - `resource`.[\[\[isComparison\]\]](https://gpuweb.github.io#dom-gpusampler-iscomparison-slot)`false`.
          - ["non-filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-non-filtering)
          - `resource`.[\[\[isFiltering\]\]](https://gpuweb.github.io#dom-gpusampler-isfiltering-slot)`false`. `resource`.[\[\[isComparison\]\]](https://gpuweb.github.io#dom-gpusampler-iscomparison-slot)`false`.
          - ["comparison"](https://gpuweb.github.io#dom-gpusamplerbindingtype-comparison)
          - `resource`.[\[\[isComparison\]\]](https://gpuweb.github.io#dom-gpusampler-iscomparison-slot)`true`.
      - [texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)
        - `resource` is either a [GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)
        - `resource` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
        - Let `textureView` be [get as texture view](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)(`resource`).
        - Let `texture` be `textureView`.[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)
        - `layoutBinding`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[viewDimension](https://gpuweb.github.io#dom-gputexturebindinglayout-viewdimension)`textureView`’s [dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
        - `layoutBinding`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)[compatible](https://gpuweb.github.io#texture-format-caps) with `textureView`’s [format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)
        - `textureView`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)[TEXTURE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-texture_binding)
        - If `layoutBinding`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[multisampled](https://gpuweb.github.io#dom-gputexturebindinglayout-multisampled)`true`, `texture`’s [sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)`1`, Otherwise `texture`’s [sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)`1`.
        - 
                If`texture`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)
`undefined`:
          - [Assert](https://infra.spec.whatwg.org/#assert) `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
          - `texture`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)`textureView`.[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
      - [storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)
        - `resource` is either a [GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)
        - `resource` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
        - Let `storageTextureView` be [get as texture view](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)(`resource`).
        - Let `texture` be `storageTextureView`.[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)
        - `layoutBinding`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[viewDimension](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-viewdimension)`storageTextureView`’s [dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
        - `layoutBinding`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[format](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-format)`storageTextureView`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)
        - `storageTextureView`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)
        - `storageTextureView`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)
        - `storageTextureView`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[swizzle](https://gpuweb.github.io#dom-gputextureviewdescriptor-swizzle)`"rgba"`.
        - 
                If`texture`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)
`undefined`:
          - [Assert](https://infra.spec.whatwg.org/#assert) `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
          - `texture`.[textureBindingViewDimension](https://gpuweb.github.io#dom-gputexture-texturebindingviewdimension)`storageTextureView`.[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
      - [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
        - `resource` is either a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)
        - Let `bufferBinding` be [get as buffer binding](https://gpuweb.github.io#abstract-opdef-get-as-buffer-binding)(`resource`).
        - `bufferBinding`.[buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)[valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
        - The bound part designated by `bufferBinding`.[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)`bufferBinding`.[size](https://gpuweb.github.io#dom-gpubufferbinding-size)
        - [effective buffer binding size](https://gpuweb.github.io#abstract-opdef-effective-buffer-binding-size)(`bufferBinding`) ≥ `layoutBinding`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)
        - If `layoutBinding`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)
          - ["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform)
            - `bufferBinding`.[buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)[usage](https://gpuweb.github.io#dom-gpubufferdescriptor-usage)[UNIFORM](https://gpuweb.github.io#dom-gpubufferusage-uniform)
            - [effective buffer binding size](https://gpuweb.github.io#abstract-opdef-effective-buffer-binding-size)(`bufferBinding`) ≤ `limits`.[maxUniformBufferBindingSize](https://gpuweb.github.io#dom-supported-limits-maxuniformbufferbindingsize)
            - `bufferBinding`.[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)`limits`.[minUniformBufferOffsetAlignment](https://gpuweb.github.io#dom-supported-limits-minuniformbufferoffsetalignment)
          - ["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)
["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage)
            - `bufferBinding`.[buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)[usage](https://gpuweb.github.io#dom-gpubufferdescriptor-usage)[STORAGE](https://gpuweb.github.io#dom-gpubufferusage-storage)
            - [effective buffer binding size](https://gpuweb.github.io#abstract-opdef-effective-buffer-binding-size)(`bufferBinding`) ≤ `limits`.[maxStorageBufferBindingSize](https://gpuweb.github.io#dom-supported-limits-maxstoragebufferbindingsize)
            - [effective buffer binding size](https://gpuweb.github.io#abstract-opdef-effective-buffer-binding-size)(`bufferBinding`) is a multiple of 4.
            - `bufferBinding`.[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)`limits`.[minStorageBufferOffsetAlignment](https://gpuweb.github.io#dom-supported-limits-minstoragebufferoffsetalignment)
      - [externalTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-externaltexture)
        - `resource` is either a [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)[GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)
        - `resource` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
        - If `resource` is a:
          - [GPUTexture](https://gpuweb.github.io#gputexture)
[GPUTextureView](https://gpuweb.github.io#gputextureview)
            - Let `view` be [get as texture view](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)(`resource`).
            - `view`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)[TEXTURE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-texture_binding)
            - `view`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)
            - `view`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)
            - `view`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)["rgba8unorm"](https://gpuweb.github.io#dom-gputextureformat-rgba8unorm)["bgra8unorm"](https://gpuweb.github.io#dom-gputextureformat-bgra8unorm)["rgba16float"](https://gpuweb.github.io#dom-gputextureformat-rgba16float)
            - `view`.[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)
    - 
            If`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
      - For each [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)`bindGroupEntry` in `descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgroupdescriptor-entries)
        - If `bindGroupEntry`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)[GPUTextureView](https://gpuweb.github.io#gputextureview)
          - Let `textureView` be `bindGroupEntry`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)
          - Let `descriptor` be `textureView`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)
          - `descriptor`.[baseArrayLayer](https://gpuweb.github.io#dom-gputextureviewdescriptor-basearraylayer)`0`.
          - `descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)`textureView`.[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)[depthOrArrayLayers](https://gpuweb.github.io#dom-gputexture-depthorarraylayers)
  3. Let `bindGroup`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpubindgroup-layout-slot)`descriptor`.[layout](https://gpuweb.github.io#dom-gpubindgroupdescriptor-layout)
  4. Let `bindGroup`.[\[\[entries\]\]](https://gpuweb.github.io#dom-gpubindgroup-entries-slot)`descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgroupdescriptor-entries)
  5. Let `bindGroup`.[\[\[usedResources\]\]](https://gpuweb.github.io#dom-gpubindgroup-usedresources-slot)
  6. For each [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)`bindingDescriptor` in `descriptor`.[entries](https://gpuweb.github.io#dom-gpubindgroupdescriptor-entries)
    1. Let `internalUsage` be the [binding usage](https://gpuweb.github.io#binding-usage) for `layoutBinding`.
    2. Each [subresource](https://gpuweb.github.io#subresource) seen by `resource` is added to [\[\[usedResources\]\]](https://gpuweb.github.io#dom-gpubindgroup-usedresources-slot)`internalUsage`.
    3. Let `bindingDescriptor`.[\[\[prevalidatedSize\]\]](https://gpuweb.github.io#dom-gpubindgroupentry-prevalidatedsize-slot)`false` if the defined [binding member](https://gpuweb.github.io#binding-member) for `layoutBinding` is [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)`layoutBinding`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)`0`, and `true` otherwise.

`resource`)

**Arguments:**

- [GPUBindingResource](https://gpuweb.github.io#typedefdef-gpubindingresource)`resource`

**Returns:** [GPUTextureView](https://gpuweb.github.io#gputextureview)

1. [Assert](https://infra.spec.whatwg.org/#assert) `resource` is either a [GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)
2. If `resource` is a:
  - [GPUTexture](https://gpuweb.github.io#gputexture)
    1. Return `resource`.[createView()](https://gpuweb.github.io#dom-gputexture-createview)
  - [GPUTextureView](https://gpuweb.github.io#gputextureview)
    1. Return `resource`.

`resource`)

**Arguments:**

- [GPUBindingResource](https://gpuweb.github.io#typedefdef-gpubindingresource)`resource`

**Returns:** [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)

1. [Assert](https://infra.spec.whatwg.org/#assert) `resource` is either a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)
2. If `resource` is a:
  - [GPUBuffer](https://gpuweb.github.io#gpubuffer)
    1. Let `bufferBinding` a new [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)
    2. Set `bufferBinding`.[buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)`resource`.
    3. Return `bufferBinding`.
  - [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)
    1. Return `resource`.

[GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)
`a`and

`b`are considered buffer-binding-aliasing if and only if all of the following are true:

- `a`.[buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)`b`.[buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)
- The range formed by `a`.[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)`a`.[size](https://gpuweb.github.io#dom-gpubufferbinding-size)`b`.[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)`b`.[size](https://gpuweb.github.io#dom-gpubufferbinding-size)[size](https://gpuweb.github.io#dom-gpubufferbinding-size)[unspecified](https://infra.spec.whatwg.org/#map-exists),
the range goes to the end of the buffer.

Note: When doing this calculation, any dynamic offsets have already been applied to the ranges.

### 8.3. `GPUPipelineLayout`

A [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[setBindGroup()](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup), and the shaders of the pipeline set by [GPURenderCommandsMixin.setPipeline](https://gpuweb.github.io#dom-gpurendercommandsmixin-setpipeline)[GPUComputePassEncoder.setPipeline](https://gpuweb.github.io#dom-gpucomputepassencoder-setpipeline)

The full binding address of a resource can be defined as a trio of:

1. shader stage mask, to which the resource is visible
2. bind group index
3. binding number

The components of this address can also be seen as the binding space of a pipeline. A [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)

\[Exposed Window ,Worker ),SecureContext interface GPUPipelineLayout GPUPipelineLayout includes GPUObjectBase 

[GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[bindGroupLayouts]]`, of type [list](https://infra.spec.whatwg.org/#list)\<[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
- The [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUPipelineLayoutDescriptor.bindGroupLayouts](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-bindgrouplayouts)
- `[[immediateSize]]`, of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- The size in bytes of the immediate data range, provided at creation in [GPUPipelineLayoutDescriptor.immediateSize](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-immediatesize)[default pipeline layout](https://gpuweb.github.io#abstract-opdef-default-pipeline-layout)).

Note: using the same [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)


[GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[GPUPipelineLayout.bindGroupLayouts](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)[GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[GPUPipelineLayout.bindGroupLayouts](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)
1. [setBindGroup](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup)(0, ...)
2. [setBindGroup](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup)(1, ...)
3. [setBindGroup](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup)(2, ...)
4. [setPipeline](https://gpuweb.github.io#dom-gpucomputepassencoder-setpipeline)
5. [dispatchWorkgroups](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups)
6. [setBindGroup](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup)(1, ...)
7. [setPipeline](https://gpuweb.github.io#dom-gpucomputepassencoder-setpipeline)
8. [dispatchWorkgroups](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups)

In this scenario, the user agent would have to re-bind the group slot 2 for the second dispatch, even though neither the [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUPipelineLayout.bindGroupLayouts](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)

Note: the expected usage of the [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)

#### 8.3.1. Pipeline Layout Creation

A [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)[GPUDevice.createPipelineLayout()](https://gpuweb.github.io#dom-gpudevice-createpipelinelayout)

dictionary GPUPipelineLayoutDescriptor GPUObjectDescriptorBase required sequence GPUBindGroupLayout bindGroupLayouts GPUSize32 immediateSize 

[GPUPipelineLayoutDescriptor](https://gpuweb.github.io#dictdef-gpupipelinelayoutdescriptor)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)

- `bindGroupLayouts`,  of type `sequence<GPUBindGroupLayout?>`
- A list of optional [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[@group](https://gpuweb.github.io/gpuweb/wgsl/#attribute-group) attribute in the [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)`N`th element corresponding
with `@group(N)`.
- `immediateSize`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32), defaulting to `0`
- The size, in bytes, of the immediate data range used by the pipeline.

- `createPipelineLayout(descriptor)`
- Creates a [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createPipelineLayout(descriptor)](https://gpuweb.github.io#dom-gpudevice-createpipelinelayout) method.Parameter Type Nullable Optional Description `descriptor`
[GPUPipelineLayoutDescriptor](https://gpuweb.github.io#dictdef-gpupipelinelayoutdescriptor)
✘ ✘ Description of the [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)
**Returns:** [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `pl` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)`descriptor`).
  2. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  3. Return `pl`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Let `limits` be `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)
  2. Let `bindGroupLayouts` be a [list](https://infra.spec.whatwg.org/#list) of `null` [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[size](https://infra.spec.whatwg.org/#list-size) equal to `limits`.[maxBindGroups](https://gpuweb.github.io#dom-supported-limits-maxbindgroups)
  3. [For each](https://infra.spec.whatwg.org/#list-iterate) `bindGroupLayout` at index `i` in `descriptor`.[bindGroupLayouts](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-bindgrouplayouts)
    1. If `bindGroupLayout` is not `null` and `bindGroupLayout`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-descriptor-slot)[entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)[empty](https://infra.spec.whatwg.org/#list-empty):
      1. Set `bindGroupLayouts`\[`i`\] to `bindGroupLayout`.
  4. Let `allEntries` be the result of concatenating `bgl`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-descriptor-slot)[entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)`null` `bgl` in `bindGroupLayouts`.
  5. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pl` and return.
    - Every non-`null` [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)`bindGroupLayouts` must be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this` and have a [\[\[exclusivePipeline\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-exclusivepipeline-slot)`null`.
    - The [size](https://infra.spec.whatwg.org/#list-size) of `descriptor`.[bindGroupLayouts](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-bindgrouplayouts)`limits`.[maxBindGroups](https://gpuweb.github.io#dom-supported-limits-maxbindgroups)
    - `allEntries` must not [exceed the binding slot limits](https://gpuweb.github.io#exceeds-the-binding-slot-limits) of `limits`.
    - `descriptor`.[immediateSize](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-immediatesize)`limits`.[maxImmediateSize](https://gpuweb.github.io#dom-supported-limits-maximmediatesize)
    - `descriptor`.[immediateSize](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-immediatesize)
  6. Set the `pl`.[\[\[bindGroupLayouts\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)`bindGroupLayouts`.
  7. Set the `pl`.[\[\[immediateSize\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-immediatesize-slot)`descriptor`.[immediateSize](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-immediatesize)

Note: two [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)[\[\[bindGroupLayouts\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[group-equivalent](https://gpuweb.github.io#group-equivalent) and their
[\[\[immediateSize\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-immediatesize-slot)

### 8.4. Example

[Create a](https://gpuweb.github.io#example-849613c6)

[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
const bindGroupLayout= gpuDevice. createBindGroupLayout({ entries: \[{ binding: 0 , visibility: GPUShaderStage. VERTEX|GPUShaderStage. FRAGMENT, buffer: {} }, { binding: 1 , visibility: GPUShaderStage. FRAGMENT, texture: {} }, { binding: 2 , visibility: GPUShaderStage. FRAGMENT, sampler: {} }\] }); const bindGroup= gpuDevice. createBindGroup({ layout: bindGroupLayout, entries: \[{ binding: 0 , resource: { buffer: buffer}, }, { binding: 1 , resource: texture}, { binding: 2 , resource: sampler}\] }); const pipelineLayout= gpuDevice. createPipelineLayout({ bindGroupLayouts: \[ bindGroupLayout\] }); 

## 9. Shader Modules

### 9.1. `GPUShaderModule`

\[Exposed Window ,Worker ),SecureContext interface GPUShaderModule Promise GPUCompilationInfo getCompilationInfo GPUShaderModule includes GPUObjectBase 

[GPUShaderModule](https://gpuweb.github.io#gpushadermodule)

#### 9.1.1. Shader Module Creation

dictionary GPUShaderModuleDescriptor GPUObjectDescriptorBase required USVString code sequence GPUShaderModuleCompilationHint compilationHints 

- `code`,  of type [USVString](https://webidl.spec.whatwg.org/#idl-USVString)
- The [WGSL](https://gpuweb.github.io/gpuweb/wgsl/) source code for the shader
module.
- `compilationHints`,  of type sequence\<[GPUShaderModuleCompilationHint](https://gpuweb.github.io#dictdef-gpushadermodulecompilationhint)>, defaulting to `[]`
- A list of [GPUShaderModuleCompilationHint](https://gpuweb.github.io#dictdef-gpushadermodulecompilationhint)
Any hint provided by an application **should** contain information about one entry point of
a pipeline that will eventually be created from the entry point.
Implementations **should** use any information present in the [GPUShaderModuleCompilationHint](https://gpuweb.github.io#dictdef-gpushadermodulecompilationhint)[createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)
Aside from type-checking, these hints are not validated in any way. NOTE:Supplying information in[compilationHints](https://gpuweb.github.io#dom-gpushadermoduledescriptor-compilationhints)
Because a single shader module can hold multiple entry points, and multiple pipelines
    can be created from a single shader module, it can be more performant for an
    implementation to do as much compilation as possible once in
    [createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)[createComputePipeline()](https://gpuweb.github.io#dom-gpudevice-createcomputepipeline)[createRenderPipeline()](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline)
Hints are only applied to the entry points they explicitly name.
    Unlike [GPUProgrammableStage.entryPoint](https://gpuweb.github.io#dom-gpuprogrammablestage-entrypoint)
Note:
Hints are not validated in an observable way, but user agents **may** surface identifiable
errors (like unknown entry point names or incompatible pipeline layouts) to developers,
for example in the browser developer console.

- `createShaderModule(descriptor)`
- Creates a [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
**Arguments:**
Arguments for the [GPUDevice.createShaderModule(descriptor)](https://gpuweb.github.io#dom-gpudevice-createshadermodule) method.Parameter Type Nullable Optional Description `descriptor`
[GPUShaderModuleDescriptor](https://gpuweb.github.io#dictdef-gpushadermoduledescriptor)
✘ ✘ Description of the [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
**Returns:** [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `sm` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)`descriptor`).
  2. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  3. Return `sm`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Let `error` be any error that results from [shader module creation](https://gpuweb.github.io/gpuweb/wgsl/#shader-module-creation) with the
WGSL source `descriptor`.[code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)`null` if no
errors occured.
  2. If any of the following requirements are unmet,
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `sm`, and return.
    - `this` must not be [lost](https://gpuweb.github.io#abstract-opdef-invalid).
    - `error` must not be a [shader-creation](https://gpuweb.github.io/gpuweb/wgsl/#shader-creation-error) [program error](https://gpuweb.github.io/gpuweb/wgsl/#program-error).
    - For each `enable` extension in `descriptor`.[code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)[GPUFeatureName](https://gpuweb.github.io#gpufeaturename)[Feature Index](https://gpuweb.github.io#feature-index)).
Note: [Uncategorized errors](https://gpuweb.github.io/gpuweb/wgsl/#uncategorized-error) cannot arise from shader module creation.
Implementations which detect such errors during shader module creation
must behave as if the shader module is valid, and defer surfacing the
error until pipeline creation.

  NOTE:User agents**should not** include detailed compiler error messages or shader text in
            the [message](https://gpuweb.github.io#dom-gpuerror-message)
[getCompilationInfo()](https://gpuweb.github.io#dom-gpushadermodule-getcompilationinfo)
**should** surface human-readable, formatted error details *to
            developers* for easier debugging (for example as a warning in the browser developer
            console, expandable to show full shader source). As shader compilation errors should be rare in production applications, user agents
            could choose to surface them *to developers* regardless of error handling ([GPU error scopes](https://gpuweb.github.io#gpu-error-scope) or [uncapturederror](https://gpuweb.github.io#eventdef-gpudevice-uncapturederror)[GPUCompilationInfo](https://gpuweb.github.io#gpucompilationinfo)

[Create a](https://gpuweb.github.io#example-91243acf)

[GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
// A simple vertex and fragment shader pair that will fill the viewport with red. const shaderSource= \` var\<private> pos : array\<vec2\<f32>, 3> = array\<vec2\<f32>, 3>( vec2(-1.0, -1.0), vec2(-1.0, 3.0), vec2(3.0, -1.0)); @vertex fn vertexMain(@builtin(vertex\_index) vertexIndex : u32) -> @builtin(position) vec4\<f32> { return vec4(pos\[vertexIndex\], 1.0, 1.0); } @fragment fn fragmentMain() -> @location(0) vec4\<f32> { return vec4(1.0, 0.0, 0.0, 1.0); } \` ; const shaderModule= gpuDevice. createShaderModule({ code: shaderSource, }); 

##### 9.1.1.1. Shader Module Compilation Hints

Shader module compilation hints are optional, additional information indicating how a given
[GPUShaderModule](https://gpuweb.github.io#gpushadermodule)

dictionary GPUShaderModuleCompilationHint required USVString entryPoint GPUPipelineLayout or GPUAutoLayoutMode layout 

- `layout`,  of type `(GPUPipelineLayout or GPUAutoLayoutMode)`
- A [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)[GPUShaderModule](https://gpuweb.github.io#gpushadermodule)[createComputePipeline()](https://gpuweb.github.io#dom-gpudevice-createcomputepipeline)[createRenderPipeline()](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline)["auto"](https://gpuweb.github.io#dom-gpuautolayoutmode-auto)[default pipeline layout](https://gpuweb.github.io#abstract-opdef-default-pipeline-layout) for the entry point associated with this hint will be used.

[createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)[createComputePipeline()](https://gpuweb.github.io#dom-gpudevice-createcomputepipeline)[createRenderPipeline()](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline)
If an application is unable to provide hint information at the time of calling
    [createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)[createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)[compilationHints](https://gpuweb.github.io#dom-gpushadermoduledescriptor-compilationhints)[GPUShaderModuleCompilationHint](https://gpuweb.github.io#dictdef-gpushadermodulecompilationhint)[createComputePipeline()](https://gpuweb.github.io#dom-gpudevice-createcomputepipeline)[createRenderPipeline()](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline)

If an author is not confident that the hint information passed to [createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)[createComputePipeline()](https://gpuweb.github.io#dom-gpudevice-createcomputepipeline)[createRenderPipeline()](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline)[createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)[createShaderModule()](https://gpuweb.github.io#dom-gpudevice-createshadermodule)

#### 9.1.2. Shader Module Compilation Information

enum GPUCompilationMessageType "error" "warning" "info" Exposed Window ,Worker ),Serializable SecureContext interface GPUCompilationMessage readonly attribute DOMString message readonly attribute GPUCompilationMessageType type readonly attribute unsigned long long lineNum readonly attribute unsigned long long linePos readonly attribute unsigned long long offset readonly attribute unsigned long long length Exposed Window ,Worker ),Serializable SecureContext interface GPUCompilationInfo readonly attribute FrozenArray GPUCompilationMessage messages 

A [GPUCompilationMessage](https://gpuweb.github.io#gpucompilationmessage)[GPUShaderModule](https://gpuweb.github.io#gpushadermodule)[code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)

[GPUCompilationMessage](https://gpuweb.github.io#gpucompilationmessage)

- `message`,  of type [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString), readonly
- The human-readable, [localizable text](https://www.w3.org/TR/i18n-glossary/#dfn-localizable-text) for this compilation message.
Note: The [message](https://gpuweb.github.io#dom-gpucompilationmessage-message)[best practices for language and direction information](https://w3c.github.io/string-meta/#bp_and-reco). This includes making use of any future standards which may
emerge regarding the reporting of string language and direction metadata.
Editorial note: At the time of this writing, no language/direction recommendation is available that provides compatibility and consistency with legacy APIs, but when there is, adopt it formally.
- `type`,  of type [GPUCompilationMessageType](https://gpuweb.github.io#enumdef-gpucompilationmessagetype), readonly
- The severity level of the message. If the [type](https://gpuweb.github.io#dom-gpucompilationmessage-type)["error"](https://gpuweb.github.io#dom-gpucompilationmessagetype-error)[shader-creation error](https://gpuweb.github.io/gpuweb/wgsl/#shader-creation-error).
- `lineNum`,  of type [unsigned long long](https://webidl.spec.whatwg.org/#idl-unsigned-long-long), readonly
- The line number in the shader [code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)[message](https://gpuweb.github.io#dom-gpucompilationmessage-message)`1` indicates the first line of the shader [code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)[line breaks](https://gpuweb.github.io/gpuweb/wgsl/#line-break).
If the [message](https://gpuweb.github.io#dom-gpucompilationmessage-message)`0` if the [message](https://gpuweb.github.io#dom-gpucompilationmessage-message)[code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)
- `linePos`,  of type [unsigned long long](https://webidl.spec.whatwg.org/#idl-unsigned-long-long), readonly
- The offset, in UTF-16 code units, from the beginning of line [lineNum](https://gpuweb.github.io#dom-gpucompilationmessage-linenum)[code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)[message](https://gpuweb.github.io#dom-gpucompilationmessage-message)[linePos](https://gpuweb.github.io#dom-gpucompilationmessage-linepos)`1` indicates the first code unit of the line.
If [message](https://gpuweb.github.io#dom-gpucompilationmessage-message)`0` if the [message](https://gpuweb.github.io#dom-gpucompilationmessage-message)[code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)
- `offset`,  of type [unsigned long long](https://webidl.spec.whatwg.org/#idl-unsigned-long-long), readonly
- The offset from the beginning of the shader [code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)[message](https://gpuweb.github.io#dom-gpucompilationmessage-message)[lineNum](https://gpuweb.github.io#dom-gpucompilationmessage-linenum)[linePos](https://gpuweb.github.io#dom-gpucompilationmessage-linepos)`0` if the [message](https://gpuweb.github.io#dom-gpucompilationmessage-message)[code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)
- `length`,  of type [unsigned long long](https://webidl.spec.whatwg.org/#idl-unsigned-long-long), readonly
- The number of UTF-16 code units in the substring that [message](https://gpuweb.github.io#dom-gpucompilationmessage-message)[length](https://gpuweb.github.io#dom-gpucompilationmessage-length)

Note: [GPUCompilationMessage](https://gpuweb.github.io#gpucompilationmessage)[lineNum](https://gpuweb.github.io#dom-gpucompilationmessage-linenum)[GPUCompilationMessage](https://gpuweb.github.io#gpucompilationmessage)[linePos](https://gpuweb.github.io#dom-gpucompilationmessage-linepos)

Note: [GPUCompilationMessage](https://gpuweb.github.io#gpucompilationmessage)[offset](https://gpuweb.github.io#dom-gpucompilationmessage-offset)[GPUCompilationMessage](https://gpuweb.github.io#gpucompilationmessage)[length](https://gpuweb.github.io#dom-gpucompilationmessage-length)`substr()` in order to retrieve the substring of the shader [code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)[message](https://gpuweb.github.io#dom-gpucompilationmessage-message)

- `getCompilationInfo()`
- Returns any messages generated during the [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
The locations, order, and contents of messages are [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined).
In particular, messages aren’t necessarily ordered by [lineNum](https://gpuweb.github.io#dom-gpucompilationmessage-linenum)
**Called on:** [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
**Returns:** [Promise](https://webidl.spec.whatwg.org/#idl-promise)[GPUCompilationInfo](https://gpuweb.github.io#gpucompilationinfo)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `contentTimeline` be the current [Content timeline](https://gpuweb.github.io#content-timeline).
  2. Let `promise` be [a new promise](https://webidl.spec.whatwg.org/#a-new-promise).
  3. Issue the `synchronization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  4. Return `promise`.
[Device timeline](https://gpuweb.github.io#device-timeline) `synchronization steps`:
  1. Let `event` occur upon the (successful or unsuccessful) completion of [shader module creation](https://gpuweb.github.io/gpuweb/wgsl/#shader-module-creation) for `this`.
  2. [Listen for timeline event](https://gpuweb.github.io#abstract-opdef-listen-for-timeline-event) `event` on `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)`contentTimeline`.
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `info` be a new [GPUCompilationInfo](https://gpuweb.github.io#gpucompilationinfo)
  2. Let `messages` be a list of any errors, warnings, or informational messages
generated during [shader module creation](https://gpuweb.github.io/gpuweb/wgsl/#shader-module-creation) for `this`, or the empty list `[]` if the device was lost.
  3. For each `message` in `messages`:
    1. Let `m` be a new [GPUCompilationMessage](https://gpuweb.github.io#gpucompilationmessage)
    2. Set `m`.[message](https://gpuweb.github.io#dom-gpucompilationmessage-message)`message`.
      - If `message` is associated with a specific substring or position
within the shader [code](https://gpuweb.github.io#dom-gpushadermoduledescriptor-code)
        1. Set `m`.[lineNum](https://gpuweb.github.io#dom-gpucompilationmessage-linenum)
        2. Set `m`.[linePos](https://gpuweb.github.io#dom-gpucompilationmessage-linepos)`m`.[lineNum](https://gpuweb.github.io#dom-gpucompilationmessage-linenum)`1` if the `message` refers to
the entire line.
        3. Set `m`.[offset](https://gpuweb.github.io#dom-gpucompilationmessage-offset)`message` refers to.
        4. Set `m`.[length](https://gpuweb.github.io#dom-gpucompilationmessage-length)`message` refers to, or 0
if `message` refers to a position
      - Otherwise:
    4. [Append](https://infra.spec.whatwg.org/#list-append) `m` to `info`.[messages](https://gpuweb.github.io#dom-gpucompilationinfo-messages)
  4. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise` with `info`.

## 10. Pipelines

A pipeline, be it [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)

Structurally, the [pipeline](https://gpuweb.github.io#pipeline) consists of a sequence of programmable stages (shaders)
and fixed-function states, such as the blending modes.

Note: Internally, depending on the target platform, the driver may convert some of the fixed-function states into shader code, and link it together with the shaders provided by the user. This linking is one of the reason the object is created as a whole.

This combination state is created as a single object
(a [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)[setPipeline()](https://gpuweb.github.io#dom-gpucomputepassencoder-setpipeline)[GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)[setPipeline()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setpipeline)

There are two ways to create pipelines:

- immediate pipeline creation
- [createComputePipeline()](https://gpuweb.github.io#dom-gpudevice-createcomputepipeline)[createRenderPipeline()](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline)
When this fails, the pipeline object will be invalid and the call will generate either a
[validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error) or an [internal error](https://gpuweb.github.io#abstract-opdef-generate-an-internal-error).
Note:
A handle object is returned immediately, but actual pipeline creation is not synchronous.
If pipeline creation takes a long time, this can incur a stall in the
[device timeline](https://gpuweb.github.io#device-timeline) at some point between the creation call and execution of the [submit()](https://gpuweb.github.io#dom-gpuqueue-submit)`setPipeline()`, at the corresponding `finish()` of that [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)[submit()](https://gpuweb.github.io#dom-gpuqueue-submit)[GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)
- async pipeline creation
- [createComputePipelineAsync()](https://gpuweb.github.io#dom-gpudevice-createcomputepipelineasync)[createRenderPipelineAsync()](https://gpuweb.github.io#dom-gpudevice-createrenderpipelineasync)`Promise` which resolves to a pipeline object when creation of the pipeline has
completed.
When this fails, the `Promise` rejects with a [GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)

`GPUPipelineError` describes a pipeline creation failure.

\[Exposed Window ,Worker ),SecureContext Serializable interface GPUPipelineError DOMException constructor optional DOMString message GPUPipelineErrorInit options readonly attribute GPUPipelineErrorReason reason dictionary GPUPipelineErrorInit required GPUPipelineErrorReason reason enum GPUPipelineErrorReason "validation" "internal" 

[GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)

- `constructor()`
- **Arguments:**Arguments for the [GPUPipelineError.constructor()](https://gpuweb.github.io#dom-gpupipelineerror-constructor) method.Parameter Type Nullable Optional Description `message`
[DOMString](https://webidl.spec.whatwg.org/#idl-DOMString)
✘ ✔ Error message of the base [DOMException](https://webidl.spec.whatwg.org/#idl-DOMException)
`options`
[GPUPipelineErrorInit](https://gpuweb.github.io#dictdef-gpupipelineerrorinit)
✘ ✘ Options specific to [GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:

[GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)

- `reason`,  of type [GPUPipelineErrorReason](https://gpuweb.github.io#enumdef-gpupipelineerrorreason), readonly
- A read-only [slot-backed attribute](https://gpuweb.github.io#slot-backed-attribute) exposing the type of error encountered in pipeline creation
as a `GPUPipelineErrorReason`:
  - `"validation"`: A [validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).
  - `"internal"`: An [internal error](https://gpuweb.github.io#abstract-opdef-generate-an-internal-error).

[GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)[serializable objects](https://html.spec.whatwg.org/multipage/structured-data.html#serializable-objects).

[serialization steps](https://html.spec.whatwg.org/multipage/structured-data.html#serialization-steps), given

`value`and

`serialized`, are:

1. Run the [DOMException](https://webidl.spec.whatwg.org/#idl-DOMException)[serialization steps](https://html.spec.whatwg.org/multipage/structured-data.html#serialization-steps) given `value` and `serialized`.

[deserialization steps](https://html.spec.whatwg.org/multipage/structured-data.html#deserialization-steps), given

`value`and

`serialized`, are:

1. Run the [DOMException](https://webidl.spec.whatwg.org/#idl-DOMException)[deserialization steps](https://html.spec.whatwg.org/multipage/structured-data.html#deserialization-steps) given `value` and `serialized`.

### 10.1. Base pipelines

enum GPUAutoLayoutMode "auto" dictionary GPUPipelineDescriptorBase GPUObjectDescriptorBase required (GPUPipelineLayout or GPUAutoLayoutMode layout 

- `layout`,  of type `(GPUPipelineLayout or GPUAutoLayoutMode)`
- The [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)["auto"](https://gpuweb.github.io#dom-gpuautolayoutmode-auto)
Note: If ["auto"](https://gpuweb.github.io#dom-gpuautolayoutmode-auto)[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)

interface mixin GPUPipelineBase NewObject GPUBindGroupLayout getBindGroupLayout unsigned long index 

[GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[layout]]`, of type `GPUPipelineLayout`
- The definition of the layout of resources which can be used with `this`.

[GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)

- `getBindGroupLayout(index)`
- Gets a [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)`index`.
**Called on:** [GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)
`this` **Arguments:**
Arguments for the [GPUPipelineBase.getBindGroupLayout(index)](https://gpuweb.github.io#dom-gpupipelinebase-getbindgrouplayout) method.Parameter Type Nullable Optional Description `index`
[unsigned long](https://webidl.spec.whatwg.org/#idl-unsigned-long)
✘ ✘ Index into the pipeline layout’s [\[\[bindGroupLayouts\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)
**Returns:** [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `layout` be a new [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
  2. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  3. Return `layout`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Let `limits` be `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)
  2. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `layout` and return.
    - `this` must be [valid](https://gpuweb.github.io#abstract-opdef-valid).
    - `index` \< `limits`.[maxBindGroups](https://gpuweb.github.io#dom-supported-limits-maxbindgroups)
  3. Initialize `layout` so it is a copy of `this`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpupipelinebase-layout-slot)[\[\[bindGroupLayouts\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)`index`\].
Note: [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[internal object](https://gpuweb.github.io#internal-object) with a new [WebGPU interface](https://gpuweb.github.io#webgpu-interface).
A new [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[WebGPU interface](https://gpuweb.github.io#webgpu-interface) is returned each time to avoid a round-trip
between the [Content timeline](https://gpuweb.github.io#content-timeline) and the [Device timeline](https://gpuweb.github.io#device-timeline).

#### 10.1.1. Default pipeline layout

A [GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)[layout](https://gpuweb.github.io#dom-gpupipelinedescriptorbase-layout)["auto"](https://gpuweb.github.io#dom-gpuautolayoutmode-auto)

Note: Default layouts are provided as a convenience for simple pipelines, but use of explicit layouts is recommended in most cases. Bind groups created from default layouts cannot be used with other pipelines, and the structure of the default layout may change when altering shaders, causing unexpected bind group creation errors.

To create a default pipeline layout for [GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)`pipeline`,
run the following [device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `groupCount` be 0.
2. Let `immediateSize` be 0.
3. Let `groupDescs` be a sequence of `device`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxBindGroups](https://gpuweb.github.io#dom-supported-limits-maxbindgroups)[GPUBindGroupLayoutDescriptor](https://gpuweb.github.io#dictdef-gpubindgrouplayoutdescriptor)
4. For each `groupDesc` in `groupDescs`:
5. For each [GPUProgrammableStage](https://gpuweb.github.io#gpuprogrammablestage)`stageDesc` in the descriptor used to create `pipeline`:
  1. Let `shaderStage` be the [GPUShaderStageFlags](https://gpuweb.github.io#typedefdef-gpushaderstageflags)`stageDesc` is used in `pipeline`.
  2. Let `entryPoint` be [get the entry point](https://gpuweb.github.io#abstract-opdef-get-the-entry-point)(`shaderStage`, `stageDesc`). [Assert](https://infra.spec.whatwg.org/#assert) `entryPoint` is not `null`.
  3. For each resource `resource` [statically used](https://gpuweb.github.io#statically-used) by `entryPoint`:
    1. Let `group` be `resource`’s "group" decoration.
    2. Let `binding` be `resource`’s "binding" decoration.
    3. Let `entry` be a new [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)
    4. Set `entry`.[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)`binding`.
    5. Set `entry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)`shaderStage`.
    6. If `resource` is for a sampler binding:
      1. Let `samplerLayout` be a new [GPUSamplerBindingLayout](https://gpuweb.github.io#dictdef-gpusamplerbindinglayout)
      2. Set `entry`.[sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)`samplerLayout`.
    7. If `resource` is for a comparison sampler binding:
      1. Let `samplerLayout` be a new [GPUSamplerBindingLayout](https://gpuweb.github.io#dictdef-gpusamplerbindinglayout)
      2. Set `samplerLayout`.[type](https://gpuweb.github.io#dom-gpusamplerbindinglayout-type)["comparison"](https://gpuweb.github.io#dom-gpusamplerbindingtype-comparison)
      3. Set `entry`.[sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)`samplerLayout`.
    8. If `resource` is for a buffer binding:
      1. Let `bufferLayout` be a new [GPUBufferBindingLayout](https://gpuweb.github.io#dictdef-gpubufferbindinglayout)
      2. Set `bufferLayout`.[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)`resource`’s [minimum buffer binding size](https://gpuweb.github.io#minimum-buffer-binding-size).
      3. If `resource` is for a read-only storage buffer:
        1. Set `bufferLayout`.[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage)
      4. If `resource` is for a storage buffer:
      5. Set `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)`bufferLayout`.
    9. If `resource` is for a sampled texture binding:
      1. Let `textureLayout` be a new [GPUTextureBindingLayout](https://gpuweb.github.io#dictdef-gputexturebindinglayout)
      2. If `resource` is a depth texture binding:
        - Set `textureLayout`.[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth)
Otherwise, if the sampled type of `resource` is:
        - `f32` and there exists a [static use](https://gpuweb.github.io#statically-used) of `resource` by `stageDesc` in a texture builtin function call that also uses a sampler
        - Set `textureLayout`.[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)
        - `f32` otherwise
        - Set `textureLayout`.[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float)
        - `i32`
        - Set `textureLayout`.[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)["sint"](https://gpuweb.github.io#dom-gputexturesampletype-sint)
        - `u32`
        - Set `textureLayout`.[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint)
      3. Set `textureLayout`.[viewDimension](https://gpuweb.github.io#dom-gputexturebindinglayout-viewdimension)`resource`’s dimension.
      4. If `resource` is for a multisampled texture:
        1. Set `textureLayout`.[multisampled](https://gpuweb.github.io#dom-gputexturebindinglayout-multisampled)`true`.
      5. Set `entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)`textureLayout`.
    10. If `resource` is for a storage texture binding:
      1. Let `storageTextureLayout` be a new [GPUStorageTextureBindingLayout](https://gpuweb.github.io#dictdef-gpustoragetexturebindinglayout)
      2. Set `storageTextureLayout`.[format](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-format)`resource`’s format.
      3. Set `storageTextureLayout`.[viewDimension](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-viewdimension)`resource`’s dimension.
      4. If the access mode is: 
        - `read`
        - Set `textureLayout`.[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)["read-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-only)
        - `write`
        - Set `textureLayout`.[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)["write-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-write-only)
        - `read_write`
        - Set `textureLayout`.[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write)
      5. Set `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)`storageTextureLayout`.
    11. Set `groupCount` to max(`groupCount`, `group` + 1).
    12. If `groupDescs`\[`group`\] has an entry `previousEntry` with [binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)`binding`:
      1. If `entry` has different [visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)`previousEntry`:
        1. Add the bits set in `entry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)`previousEntry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)
      2. If `resource` is for a buffer binding and `entry` has greater [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)`previousEntry`:
        1. Set `previousEntry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)`entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)
      3. If `resource` is a sampled texture binding and `entry` has different [texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)`previousEntry` and both `entry` and `previousEntry` have [texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float)
        1. Set `previousEntry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)
      4. If any other property is unequal between `entry` and `previousEntry`:
        1. Return `null` (which will cause the creation of the pipeline to fail).
      5. If `resource` is a storage texture binding, `entry`.`storageTexture`.[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write)`previousEntry`.`storageTexture`.[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)["write-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-write-only)`previousEntry`.`storageTexture`.[format](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-format)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write)[§ 26.1.1 Plain color formats](https://gpuweb.github.io#plain-color-formats) table:
        1. Set `previousEntry`.`storageTexture`.[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write)
Otherwise: 
        1. Append `entry` to `groupDescs`\[`group`\].
  4. If there is an immediate data variable that is [statically used](https://gpuweb.github.io#statically-used) by `entryPoint` ([assert](https://infra.spec.whatwg.org/#assert) there is at most one):
    1. Let `T` be the [store type](https://gpuweb.github.io/gpuweb/wgsl/#store-type) of the immediate data variable.
    2. Set `immediateSize` to max(`immediateSize`, [roundUp](https://gpuweb.github.io/gpuweb/wgsl/#roundup)(4, [SizeOf](https://gpuweb.github.io/gpuweb/wgsl/#sizeof)(`T`))).
6. Let `groupLayouts` be a new [list](https://infra.spec.whatwg.org/#list).
7. For each `i` from 0 to `groupCount` - 1, inclusive:
  1. Let `groupDesc` be `groupDescs`\[`i`\].
  2. Let `bindGroupLayout` be the result of calling `device`.[createBindGroupLayout()](https://gpuweb.github.io#dom-gpudevice-createbindgrouplayout)`groupDesc`).
  3. Set `bindGroupLayout`.[\[\[exclusivePipeline\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-exclusivepipeline-slot)`pipeline`.
  4. Append `bindGroupLayout` to `groupLayouts`.
8. Let `desc` be a new [GPUPipelineLayoutDescriptor](https://gpuweb.github.io#dictdef-gpupipelinelayoutdescriptor)
9. Set `desc`.[bindGroupLayouts](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-bindgrouplayouts)`groupLayouts`.
10. Set `desc`.[immediateSize](https://gpuweb.github.io#dom-gpupipelinelayoutdescriptor-immediatesize)`immediateSize`.
11. Return `device`.[createPipelineLayout()](https://gpuweb.github.io#dom-gpudevice-createpipelinelayout)`desc`).

#### 10.1.2. `GPUProgrammableStage`

A [GPUProgrammableStage](https://gpuweb.github.io#gpuprogrammablestage)[GPUShaderModule](https://gpuweb.github.io#gpushadermodule)[pipeline](https://gpuweb.github.io#pipeline).
Entry point names follow the rules defined in [WGSL identifier comparison](https://gpuweb.github.io/gpuweb/wgsl/#identifier-comparison).

dictionary GPUProgrammableStage required GPUShaderModule module USVString entryPoint record USVString GPUPipelineConstantValue constants typedef double GPUPipelineConstantValue 

[GPUProgrammableStage](https://gpuweb.github.io#gpuprogrammablestage)

- `module`,  of type [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
- The [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)
- `entryPoint`,  of type [USVString](https://webidl.spec.whatwg.org/#idl-USVString)
- The name of the function in [module](https://gpuweb.github.io#dom-gpuprogrammablestage-module)
NOTE: Since the [entryPoint](https://gpuweb.github.io#dom-gpuprogrammablestage-entrypoint)[GPUProgrammableStage](https://gpuweb.github.io#gpuprogrammablestage)[get the entry point](https://gpuweb.github.io#abstract-opdef-get-the-entry-point)" algorithm to determine which entry point
it refers to.
- `constants`,  of type record\<[USVString](https://webidl.spec.whatwg.org/#idl-USVString), [GPUPipelineConstantValue](https://gpuweb.github.io#typedefdef-gpupipelineconstantvalue)>, defaulting to `{}`
- Specifies the values of [pipeline-overridable](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable) constants in the shader module [module](https://gpuweb.github.io#dom-gpuprogrammablestage-module)
Each such [pipeline-overridable](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable) constant is uniquely identified by a single [pipeline-overridable constant identifier string](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable-constant-identifier-string), representing the [pipeline constant ID](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-constant-id) of the constant if its declaration specifies one, and otherwise the
constant’s identifier name.
The key of each key-value pair must equal the
[identifier string](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable-constant-identifier-string) of one such constant, with the comparison performed
according to the rules for [WGSL identifier comparison](https://gpuweb.github.io/gpuweb/wgsl/#identifier-comparison).
When the pipeline is executed, that constant will have the specified value.
Values are specified as `GPUPipelineConstantValue`, which is a [double](https://webidl.spec.whatwg.org/#idl-double)[to WGSL type](https://gpuweb.github.io#abstract-opdef-to-wgsl-type) of the pipeline-overridable constant (`bool`/`i32`/`u32`/`f32`/`f16`).
If conversion fails, a validation error is generated.
@id ( 0 ) override has\_point\_light : bool= true ; // Algorithmic control. @id ( 1200 ) override specular\_param : f32= 2.3 ; // Numeric control. @id ( 1300 ) override gain : f32; // Must be overridden. override width : f32= 0.0 ; // Specifed at the API level // using the name "width". override depth : f32; // Specifed at the API level // using the name "depth". // Must be overridden. override height = 2 \* depth ; // The default value // (if not set at the API level), // depends on another // overridable constant. Corresponding JavaScript code, providing only the overrides which are required (have no defaults): { // ... constants: { 1300 : 2.0 , // "gain" depth: - 1 , // "depth" } } Corresponding JavaScript code, overriding all constants: { // ... constants: { 0 : false , // "has\_point\_light" 1200 : 3.0 , // "specular\_param" 1300 : 2.0 , // "gain" width: 20 , // "width" depth: - 1 , // "depth" height: 15 , // "height" } }

[GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage)
`stage`,

[GPUProgrammableStage](https://gpuweb.github.io#gpuprogrammablestage)
`descriptor`), run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. If `descriptor`.[entryPoint](https://gpuweb.github.io#dom-gpuprogrammablestage-entrypoint)[provided](https://infra.spec.whatwg.org/#map-exists):
  1. If `descriptor`.[module](https://gpuweb.github.io#dom-gpuprogrammablestage-module)`descriptor`.[entryPoint](https://gpuweb.github.io#dom-gpuprogrammablestage-entrypoint)`stage`,
return that entry point.
Otherwise, return `null`.
Otherwise: 
  1. If there is exactly one entry point in `descriptor`.[module](https://gpuweb.github.io#dom-gpuprogrammablestage-module)`stage`, return that entry point.
Otherwise, return `null`.

`stage`,

`descriptor`,

`layout`,

`device`)

**Arguments:**

- [GPUShaderStage](https://gpuweb.github.io#namespacedef-gpushaderstage)`stage`
- [GPUProgrammableStage](https://gpuweb.github.io#gpuprogrammablestage)`descriptor`
- [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)`layout`
- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`

All of the requirements in the following steps `must` be met.
    If any are unmet, return `false`; otherwise, return `true`.

1. `descriptor`.[module](https://gpuweb.github.io#dom-gpuprogrammablestage-module)`must` be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `device`.
2. Let `entryPoint` be [get the entry point](https://gpuweb.github.io#abstract-opdef-get-the-entry-point)(`stage`, `descriptor`).
3. `entryPoint must` not be `null`.
4. For each `binding` that is [statically used](https://gpuweb.github.io#statically-used) by `entryPoint`:
  - [validating shader binding](https://gpuweb.github.io#abstract-opdef-validating-shader-binding)(`binding`, `layout`) `must` return `true`.
5. For each call `call` to a texture builtin function in any of the [functions in the shader stage](https://gpuweb.github.io/gpuweb/wgsl/#functions-in-a-shader-stage) rooted at `entryPoint`:
  1. Let `textureBinding` be the texture binding used in `call`.
  2. If `textureBinding` is of type [sampled texture](https://gpuweb.github.io/gpuweb/wgsl/#type-sampled-texture) or [depth texture](https://gpuweb.github.io/gpuweb/wgsl/#type-depth-texture) and `call` uses a sampler binding `samplerBinding` of type `sampler` (excluding `sampler_comparison`):
    1. Let `texture` be the [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`textureBinding`.
    2. Let `sampler` be the [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`samplerBinding`.
    3. If `sampler`.[type](https://gpuweb.github.io#dom-gpusamplerbindinglayout-type)["filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-filtering)`texture`.[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)`must` be ["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)
Note: ["comparison"](https://gpuweb.github.io#dom-gpusamplerbindingtype-comparison)["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth)`texture_depth_*` bindings.
  3. 
        If`device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
    1. If `call` is a call to `textureLoad`, `textureBinding` must not be of type [depth texture](https://gpuweb.github.io/gpuweb/wgsl/#type-depth-texture).
    2. If `call` uses a sampler binding `samplerBinding` and `textureBinding` is of type [depth texture](https://gpuweb.github.io/gpuweb/wgsl/#type-depth-texture), `samplerBinding` must be of `sampler_comparison` type.
6. For each `key` → `value` in `descriptor`.[constants](https://gpuweb.github.io#dom-gpuprogrammablestage-constants)
  1. `key must` equal the [pipeline-overridable constant identifier string](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable-constant-identifier-string) of
some [pipeline-overridable](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable) constant defined in the shader module `descriptor`.[module](https://gpuweb.github.io#dom-gpuprogrammablestage-module)[WGSL identifier comparison](https://gpuweb.github.io/gpuweb/wgsl/#identifier-comparison).
The pipeline-overridable constant is *not* required to be [statically used](https://gpuweb.github.io#statically-used) by `entryPoint`.
Let the type of that constant be `T`.
  2. Converting the IDL value `value` [to WGSL type](https://gpuweb.github.io#abstract-opdef-to-wgsl-type) `T must` not throw a [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)
7. For each [pipeline-overridable constant identifier string](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable-constant-identifier-string) `key` which is [statically used](https://gpuweb.github.io#statically-used) by `entryPoint`:
  - If the pipeline-overridable constant identified by `key` [does not have a default value](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable-constant-default-value), `descriptor`.[constants](https://gpuweb.github.io#dom-gpuprogrammablestage-constants)`must` [contain](https://infra.spec.whatwg.org/#map-exists) `key`.
8. If there is an immediate data variable that is [statically used](https://gpuweb.github.io#statically-used) by `entryPoint` ([assert](https://infra.spec.whatwg.org/#assert) there is at most one):
  - Let `T` be the [store type](https://gpuweb.github.io/gpuweb/wgsl/#store-type) of the immediate data variable.
  - Let `requiredSize` be [SizeOf](https://gpuweb.github.io/gpuweb/wgsl/#sizeof)(`T`).
  - `layout`.[\[\[immediateSize\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-immediatesize-slot)`requiredSize`.
9. [Pipeline-creation](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-creation-error) [program errors](https://gpuweb.github.io/gpuweb/wgsl/#program-error) `must` not
result from the rules of the [\[WGSL\]](https://gpuweb.github.io#biblio-wgsl) specification.
10. 
      If`device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
  1. Let `sum` be 0.
  2. For each unique texture or external texture binding `textureBinding` that is used in any call to a texture builtin in any of the [functions in the shader stage](https://gpuweb.github.io/gpuweb/wgsl/#functions-in-a-shader-stage) rooted at `entryPoint`:
    1. Let `samplerBindings` be the set of sampler bindings used together with `textureBinding` in any call to a texture builtin in any of the [functions in the shader stage](https://gpuweb.github.io/gpuweb/wgsl/#functions-in-a-shader-stage) rooted at `entryPoint`.
    2. Let `numPairs` be `max(1, number of elements of` .`samplerBindings`)
    3. If `textureBinding` is an external texture binding:
      1. Let `numPairs` be `1 + 3 *` .`numPairs`
    4. Let `sum` be `sum` + `numPairs`
  3. `sum must` be ≤ `device`.limits.[maxSampledTexturesPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxsampledtexturespershaderstage)
  4. `sum must` be ≤ `device`.limits.[maxSamplersPerShaderStage](https://gpuweb.github.io#dom-supported-limits-maxsamplerspershaderstage)

`variable`,

`layout`)

**Arguments:**

- shader binding declaration `variable`, a module-scope variable declaration reflected from a shader module
- [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)`layout`

Let `bindGroup` be the bind group index, and `bindIndex` be the binding index,
    of the shader binding declaration `variable`.

Return `true` if all of the following conditions are satisfied:

- `layout`.[\[\[bindGroupLayouts\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)`bindGroup`\] contains
a [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`entry` whose `entry`.[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)`bindIndex`.
- If the defined [binding member](https://gpuweb.github.io#binding-member) for `entry` is:
  - [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
  - If `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)
    - ["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform)
    - `variable` is declared with address space `uniform`.
    - ["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)
    - `variable` is declared with address space `storage` and access mode `read_write`.
    - ["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage)
    - `variable` is declared with address space `storage` and access mode `read`.
If `entry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)`0`,
then it must be at least the [minimum buffer binding size](https://gpuweb.github.io#minimum-buffer-binding-size) for the associated
buffer binding variable in the shader.
  - [sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)
  - If `entry`.[sampler](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-sampler)[type](https://gpuweb.github.io#dom-gpusamplerbindinglayout-type)
    - ["filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-filtering)
["non-filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-non-filtering)
    - `variable` has type `sampler`.
    - ["comparison"](https://gpuweb.github.io#dom-gpusamplerbindingtype-comparison)
    - `variable` has type `sampler_comparison`.
  - [texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)
  - If, and only if,
`entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[multisampled](https://gpuweb.github.io#dom-gputexturebindinglayout-multisampled)`true`, `variable` has type `texture_multisampled_2d<T>` or `texture_depth_multisampled_2d<T>`.
If `entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)
    - ["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)
["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float)
["sint"](https://gpuweb.github.io#dom-gputexturesampletype-sint)
["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint)
    - `variable` has one of the types:
      - `texture_1d<T>`
      - `texture_2d<T>`
      - `texture_2d_array<T>`
      - `texture_cube<T>`
      - `texture_cube_array<T>`
      - `texture_3d<T>`
      - `texture_multisampled_2d<T>`
If `entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)
      - ["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)
["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float)
      - The sampled type `T` is `f32`.
      - ["sint"](https://gpuweb.github.io#dom-gputexturesampletype-sint)
      - The sampled type `T` is `i32`.
      - ["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint)
      - The sampled type `T` is `u32`.
    - ["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth)
    - `variable` has one of the types:
      - `texture_2d<T>`
      - `texture_2d_array<T>`
      - `texture_cube<T>`
      - `texture_cube_array<T>`
      - `texture_multisampled_2d<T>`
      - `texture_depth_2d`
      - `texture_depth_2d_array`
      - `texture_depth_cube`
      - `texture_depth_cube_array`
      - `texture_depth_multisampled_2d`
where the sampled type `T` is `f32`.
If `entry`.[texture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-texture)[viewDimension](https://gpuweb.github.io#dom-gputexturebindinglayout-viewdimension)
    - ["1d"](https://gpuweb.github.io#dom-gputextureviewdimension-1d)
    - `variable` has type `texture_1d<T>`.
    - ["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)
    - `variable` has type `texture_2d<T>` or `texture_multisampled_2d<T>`.
    - ["2d-array"](https://gpuweb.github.io#dom-gputextureviewdimension-2d-array)
    - `variable` has type `texture_2d_array<T>`.
    - ["cube"](https://gpuweb.github.io#dom-gputextureviewdimension-cube)
    - `variable` has type `texture_cube<T>`.
    - ["cube-array"](https://gpuweb.github.io#dom-gputextureviewdimension-cube-array)
    - `variable` has type `texture_cube_array<T>`.
    - ["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
    - `variable` has type `texture_3d<T>`.
  - [storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)
  - If `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[viewDimension](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-viewdimension)
    - ["1d"](https://gpuweb.github.io#dom-gputextureviewdimension-1d)
    - `variable` has type `texture_storage_1d<T, A>`.
    - ["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)
    - `variable` has type `texture_storage_2d<T, A>`.
    - ["2d-array"](https://gpuweb.github.io#dom-gputextureviewdimension-2d-array)
    - `variable` has type `texture_storage_2d_array<T, A>`.
    - ["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
    - `variable` has type `texture_storage_3d<T, A>`.
If `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)
    - ["write-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-write-only)
    - The access mode `A` is `write`.
    - ["read-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-only)
    - The access mode `A` is `read`.
    - ["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write)
    - The access mode `A` is `read_write` or `write`.
The texel format `T` equals `entry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[format](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-format)

`var`is computed as follows:

1. Let `T` be the [store type](https://gpuweb.github.io/gpuweb/wgsl/#store-type) of `var`.
2. If `T` is a [runtime-sized](https://gpuweb.github.io/gpuweb/wgsl/#runtime-sized) array, or contains a runtime-sized array, replace
that `array<E>` with `array<E, 1>`.
Note: This ensures there’s always enough memory for one element, which allows array indices to be clamped to the length of the array resulting in an in-memory access.
3. If `T` is a [runtime-sized](https://gpuweb.github.io/gpuweb/wgsl/#runtime-sized) buffer, then
  1. Find all `bufferView` and `bufferArrayView` built-in function calls where the buffer
variable is the originating variable for the pointer argument.
Let the template parameter be `RT`.
  2. Return the maximum value of [MinTypeSize](https://gpuweb.github.io/gpuweb/wgsl/#mintypesize)(`RT`) among all the calls.
Note: This ensures there’s always enough memory for a minimally sized `RT` (with `offset`=0).
4. Return [SizeOf](https://gpuweb.github.io/gpuweb/wgsl/#sizeof)(`T`).

Note: Enforcing this lower bound ensures reads and writes via the buffer variable only access memory locations within the bound region of the buffer.

[pipeline-overridable](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-overridable)constant, shader stage input, or shader stage output is considered to be statically used by an entry point if it is present in the

[interface of the shader stage](https://gpuweb.github.io/gpuweb/wgsl/#interface-of-a-shader)for that entry point.

### 10.2. `GPUComputePipeline`

A [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[pipeline](https://gpuweb.github.io#pipeline) that controls the compute shader stage,
and can be used in [GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)

Compute inputs and outputs are all contained in the bindings,
according to the given [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)["write-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-write-only)["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write)

Stages of a compute [pipeline](https://gpuweb.github.io#pipeline):

1. Compute shader

\[Exposed Window ,Worker ),SecureContext interface GPUComputePipeline GPUComputePipeline includes GPUObjectBase GPUComputePipeline includes GPUPipelineBase 

#### 10.2.1. Compute Pipeline Creation

A [GPUComputePipelineDescriptor](https://gpuweb.github.io#dictdef-gpucomputepipelinedescriptor)[pipeline](https://gpuweb.github.io#pipeline). See
[§ 23.1 Computing](https://gpuweb.github.io#computing-operations) for additional details.

dictionary GPUComputePipelineDescriptor GPUPipelineDescriptorBase required GPUProgrammableStage compute 

[GPUComputePipelineDescriptor](https://gpuweb.github.io#dictdef-gpucomputepipelinedescriptor)

- `compute`,  of type [GPUProgrammableStage](https://gpuweb.github.io#gpuprogrammablestage)
- Describes the compute shader entry point of the [pipeline](https://gpuweb.github.io#pipeline).

- `createComputePipeline(descriptor)`
- Creates a [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[immediate pipeline creation](https://gpuweb.github.io#immediate-pipeline-creation).
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createComputePipeline(descriptor)](https://gpuweb.github.io#dom-gpudevice-createcomputepipeline) method.Parameter Type Nullable Optional Description `descriptor`
[GPUComputePipelineDescriptor](https://gpuweb.github.io#dictdef-gpucomputepipelinedescriptor)
✘ ✘ Description of the [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
**Returns:** [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `pipeline` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)`descriptor`).
  2. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  3. Return `pipeline`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Let `layout` be a new [default pipeline layout](https://gpuweb.github.io#abstract-opdef-default-pipeline-layout) for `pipeline` if `descriptor`.[layout](https://gpuweb.github.io#dom-gpupipelinedescriptorbase-layout)["auto"](https://gpuweb.github.io#dom-gpuautolayoutmode-auto)`descriptor`.[layout](https://gpuweb.github.io#dom-gpupipelinedescriptorbase-layout)
  2. All of the requirements in the following steps `must` be met.
If any are unmet, [generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pipeline` and return.
    1. `layout must` be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    2. [validating GPUProgrammableStage](https://gpuweb.github.io#abstract-opdef-validating-gpuprogrammablestage)([COMPUTE](https://gpuweb.github.io#dom-gpushaderstage-compute)`descriptor`.[compute](https://gpuweb.github.io#dom-gpucomputepipelinedescriptor-compute)`layout`, `this`) `must` succeed.
    3. Let `entryPoint` be [get the entry point](https://gpuweb.github.io#abstract-opdef-get-the-entry-point)([COMPUTE](https://gpuweb.github.io#dom-gpushaderstage-compute)`descriptor`.[compute](https://gpuweb.github.io#dom-gpucomputepipelinedescriptor-compute)
[Assert](https://infra.spec.whatwg.org/#assert) `entryPoint` is not `null`.
    4. Let `workgroupStorageUsed` be the sum of [roundUp](https://gpuweb.github.io/gpuweb/wgsl/#roundup)(16, [SizeOf](https://gpuweb.github.io/gpuweb/wgsl/#sizeof)(`T`)) over each
type `T` of all variables with address space "[workgroup](https://gpuweb.github.io/gpuweb/wgsl/#address-spaces-workgroup)" [statically used](https://gpuweb.github.io#statically-used) by `entryPoint`.
`workgroupStorageUsed must` be ≤ `device`.limits.[maxComputeWorkgroupStorageSize](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupstoragesize)
    5. `entryPoint must` use ≤ `device`.limits.[maxComputeInvocationsPerWorkgroup](https://gpuweb.github.io#dom-supported-limits-maxcomputeinvocationsperworkgroup)
    6. Each component of `entryPoint`’s `workgroup_size` attribute `must` be ≤ the corresponding component in
\[`device`.limits.[maxComputeWorkgroupSizeX](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizex)`device`.limits.[maxComputeWorkgroupSizeY](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizey)`device`.limits.[maxComputeWorkgroupSizeZ](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsizez)
  3. If any [pipeline-creation](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-creation-error) [uncategorized errors](https://gpuweb.github.io/gpuweb/wgsl/#uncategorized-error) result from the implementation of pipeline creation, [generate an internal error](https://gpuweb.github.io#abstract-opdef-generate-an-internal-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pipeline` and return.
Note:
Even if the implementation detected [uncategorized errors](https://gpuweb.github.io/gpuweb/wgsl/#uncategorized-error) in shader module
creation, the error is surfaced here.
  4. Set `pipeline`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpupipelinebase-layout-slot)`layout`.
- `createComputePipelineAsync(descriptor)`
- Creates a [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[async pipeline creation](https://gpuweb.github.io#async-pipeline-creation).
The returned [Promise](https://webidl.spec.whatwg.org/#idl-promise)
If pipeline creation fails, the returned [Promise](https://webidl.spec.whatwg.org/#idl-promise)[GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)[GPUError](https://gpuweb.github.io#gpuerror)
Note: Use of this method is preferred whenever possible, as it prevents blocking the
[queue timeline](https://gpuweb.github.io#queue-timeline) work on pipeline compilation.
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createComputePipelineAsync(descriptor)](https://gpuweb.github.io#dom-gpudevice-createcomputepipelineasync) method.Parameter Type Nullable Optional Description `descriptor`
[GPUComputePipelineDescriptor](https://gpuweb.github.io#dictdef-gpucomputepipelinedescriptor)
✘ ✘ Description of the [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
**Returns:** [Promise](https://webidl.spec.whatwg.org/#idl-promise)[GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `contentTimeline` be the current [Content timeline](https://gpuweb.github.io#content-timeline).
  2. Let `promise` be [a new promise](https://webidl.spec.whatwg.org/#a-new-promise).
  3. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  4. Return `promise`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Let `pipeline` be a new [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)`this`.[createComputePipeline()](https://gpuweb.github.io#dom-gpudevice-createcomputepipeline)`descriptor`,
except capturing any errors as `error`, rather than dispatching them to the device.
  2. Let `event` occur upon the (successful or unsuccessful) completion of [pipeline creation](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-creation) for `pipeline`.
  3. [Listen for timeline event](https://gpuweb.github.io#abstract-opdef-listen-for-timeline-event) `event` on `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[device timeline](https://gpuweb.github.io#device-timeline) of `this`.
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. If `pipeline` is [valid](https://gpuweb.github.io#abstract-opdef-valid) or `this` is [lost](https://gpuweb.github.io#abstract-opdef-invalid):
    1. Issue the following steps on `contentTimeline`:
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
      1. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise` with `pipeline`.
    2. Return.
Note: No errors are generated from a device which is lost.
See [§ 22 Errors & Debugging](https://gpuweb.github.io#errors-and-debugging).
  2. If `pipeline` is [invalid](https://gpuweb.github.io#abstract-opdef-invalid) and `error` is an [internal error](https://gpuweb.github.io#abstract-opdef-generate-an-internal-error),
issue the following steps on `contentTimeline`, and return.
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `promise` with a [GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)[reason](https://gpuweb.github.io#dom-gpupipelineerrorinit-reason)["internal"](https://gpuweb.github.io#dom-gpupipelineerrorreason-internal)
  3. If `pipeline` is [invalid](https://gpuweb.github.io#abstract-opdef-invalid) and `error` is a [validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error),
issue the following steps on `contentTimeline`, and return.
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `promise` with a [GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)[reason](https://gpuweb.github.io#dom-gpupipelineerrorinit-reason)["validation"](https://gpuweb.github.io#dom-gpupipelineerrorreason-validation)

[Creating a simple](https://gpuweb.github.io#example-24ae760c)

[GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
const computePipeline= gpuDevice. createComputePipeline({ layout: pipelineLayout, compute: { module: computeShaderModule, entryPoint: 'computeMain' , } }); 

### 10.3. `GPURenderPipeline`

A [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[pipeline](https://gpuweb.github.io#pipeline) that controls the vertex
and fragment shader stages, and can be used in [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)

Render [pipeline](https://gpuweb.github.io#pipeline) inputs are:

- bindings, according to the given [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)
- vertex and index buffers, described by [GPUVertexState](https://gpuweb.github.io#dictdef-gpuvertexstate)
- the color attachments, described by [GPUColorTargetState](https://gpuweb.github.io#dictdef-gpucolortargetstate)
- optionally, the depth-stencil attachment, described by [GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)

Render [pipeline](https://gpuweb.github.io#pipeline) outputs are:

- [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)
- [storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)["write-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-write-only)["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write)
- the color attachments, described by [GPUColorTargetState](https://gpuweb.github.io#dictdef-gpucolortargetstate)
- optionally, depth-stencil attachment, described by [GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)

A render [pipeline](https://gpuweb.github.io#pipeline) is comprised of the following render stages:

1. Vertex fetch, controlled by [GPUVertexState.buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)
2. Vertex shader, controlled by [GPUVertexState](https://gpuweb.github.io#dictdef-gpuvertexstate)
3. Primitive assembly, controlled by [GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate)
4. Rasterization, controlled by [GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate)[GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)[GPUMultisampleState](https://gpuweb.github.io#dictdef-gpumultisamplestate)
5. Fragment shader, controlled by [GPUFragmentState](https://gpuweb.github.io#dictdef-gpufragmentstate)
6. Stencil test and operation, controlled by [GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)
7. Depth test and write, controlled by [GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)
8. Output merging, controlled by [GPUFragmentState.targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)

\[Exposed Window ,Worker ),SecureContext interface GPURenderPipeline GPURenderPipeline includes GPUObjectBase GPURenderPipeline includes GPUPipelineBase 

[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[descriptor]]`, of type [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)
- The [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)
All optional fields of [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)
- `[[writesDepth]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- True if the pipeline writes to the depth component of the depth/stencil attachment
- `[[writesStencil]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- True if the pipeline writes to the stencil component of the depth/stencil attachment

#### 10.3.1. Render Pipeline Creation

A [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)[pipeline](https://gpuweb.github.io#pipeline) by configuring each
of the [render stages](https://gpuweb.github.io#render-stages). See [§ 23.2 Rendering](https://gpuweb.github.io#rendering-operations) for additional details.

dictionary GPURenderPipelineDescriptor GPUPipelineDescriptorBase required GPUVertexState vertex GPUPrimitiveState primitive GPUDepthStencilState depthStencil GPUMultisampleState multisample GPUFragmentState fragment 

[GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)

- `vertex`,  of type [GPUVertexState](https://gpuweb.github.io#dictdef-gpuvertexstate)
- Describes the vertex shader entry point of the [pipeline](https://gpuweb.github.io#pipeline) and its input buffer layouts.
- `primitive`,  of type [GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate), defaulting to `{}`
- Describes the primitive-related properties of the [pipeline](https://gpuweb.github.io#pipeline).
- `depthStencil`,  of type [GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)
- Describes the optional depth-stencil properties, including the testing, operations, and bias.
- `multisample`,  of type [GPUMultisampleState](https://gpuweb.github.io#dictdef-gpumultisamplestate), defaulting to `{}`
- Describes the multi-sampling properties of the [pipeline](https://gpuweb.github.io#pipeline).
- `fragment`,  of type [GPUFragmentState](https://gpuweb.github.io#dictdef-gpufragmentstate)
- Describes the fragment shader entry point of the [pipeline](https://gpuweb.github.io#pipeline) and its output colors. If
not [provided](https://infra.spec.whatwg.org/#map-exists), the [§ 23.2.8 No Color Output](https://gpuweb.github.io#no-color-output) mode is enabled.

- `createRenderPipeline(descriptor)`
- Creates a [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[immediate pipeline creation](https://gpuweb.github.io#immediate-pipeline-creation).
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createRenderPipeline(descriptor)](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline) method.Parameter Type Nullable Optional Description `descriptor`
[GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)
✘ ✘ Description of the [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
**Returns:** [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)[provided](https://infra.spec.whatwg.org/#map-exists):
    1. [For each](https://infra.spec.whatwg.org/#list-iterate) non-`null` `colorState` of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)
      1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of `colorState`.[format](https://gpuweb.github.io#dom-gpucolortargetstate-format)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  2. If `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)[provided](https://infra.spec.whatwg.org/#map-exists):
    1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)[format](https://gpuweb.github.io#dom-gpudepthstencilstate-format)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  3. Let `pipeline` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)`descriptor`).
  4. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  5. Return `pipeline`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Let `layout` be a new [default pipeline layout](https://gpuweb.github.io#abstract-opdef-default-pipeline-layout) for `pipeline` if `descriptor`.[layout](https://gpuweb.github.io#dom-gpupipelinedescriptorbase-layout)["auto"](https://gpuweb.github.io#dom-gpuautolayoutmode-auto)`descriptor`.[layout](https://gpuweb.github.io#dom-gpupipelinedescriptorbase-layout)
  2. All of the requirements in the following steps `must` be met.
If any are unmet, [generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pipeline`, and return.
    1. `layout must` be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    2. [validating GPURenderPipelineDescriptor](https://gpuweb.github.io#abstract-opdef-validating-gpurenderpipelinedescriptor)(`descriptor`, `layout`, `this`) must succeed.
    3. Let `vertexBufferCount` be the index of the last non-null entry in `descriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)
    4. `layout`.[\[\[bindGroupLayouts\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)[size](https://infra.spec.whatwg.org/#list-size) + `vertexBufferCount` must be ≤ `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxBindGroupsPlusVertexBuffers](https://gpuweb.github.io#dom-supported-limits-maxbindgroupsplusvertexbuffers)
  3. If any [pipeline-creation](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-creation-error) [uncategorized errors](https://gpuweb.github.io/gpuweb/wgsl/#uncategorized-error) result from the implementation of pipeline creation, [generate an internal error](https://gpuweb.github.io#abstract-opdef-generate-an-internal-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pipeline` and return.
Note:
Even if the implementation detected [uncategorized errors](https://gpuweb.github.io/gpuweb/wgsl/#uncategorized-error) in shader module
creation, the error is surfaced here.
  4. Set `pipeline`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)`descriptor`.
  5. Set `pipeline`.[\[\[writesDepth\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesdepth-slot)
  6. Set `pipeline`.[\[\[writesStencil\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesstencil-slot)
  7. Let `depthStencil` be `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)
  8. If `depthStencil` is not null:
    1. If `depthStencil`.[depthWriteEnabled](https://gpuweb.github.io#dom-gpudepthstencilstate-depthwriteenabled)[provided](https://infra.spec.whatwg.org/#map-exists):
      1. Set `pipeline`.[\[\[writesDepth\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesdepth-slot)`depthStencil`.[depthWriteEnabled](https://gpuweb.github.io#dom-gpudepthstencilstate-depthwriteenabled)
    2. If `depthStencil`.[stencilWriteMask](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilwritemask)
      1. Let `stencilFront` be `depthStencil`.[stencilFront](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilfront)
      2. Let `stencilBack` be `depthStencil`.[stencilBack](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilback)
      3. Let `cullMode` be `descriptor`.[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[cullMode](https://gpuweb.github.io#dom-gpuprimitivestate-cullmode)
      4. If `cullMode` is not ["front"](https://gpuweb.github.io#dom-gpucullmode-front)`stencilFront`.[passOp](https://gpuweb.github.io#dom-gpustencilfacestate-passop)`stencilFront`.[depthFailOp](https://gpuweb.github.io#dom-gpustencilfacestate-depthfailop)`stencilFront`.[failOp](https://gpuweb.github.io#dom-gpustencilfacestate-failop)["keep"](https://gpuweb.github.io#dom-gpustenciloperation-keep)
        1. Set `pipeline`.[\[\[writesStencil\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesstencil-slot)
      5. If `cullMode` is not ["back"](https://gpuweb.github.io#dom-gpucullmode-back)`stencilBack`.[passOp](https://gpuweb.github.io#dom-gpustencilfacestate-passop)`stencilBack`.[depthFailOp](https://gpuweb.github.io#dom-gpustencilfacestate-depthfailop)`stencilBack`.[failOp](https://gpuweb.github.io#dom-gpustencilfacestate-failop)["keep"](https://gpuweb.github.io#dom-gpustenciloperation-keep)
        1. Set `pipeline`.[\[\[writesStencil\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesstencil-slot)
  9. Set `pipeline`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpupipelinebase-layout-slot)`layout`.
- `createRenderPipelineAsync(descriptor)`
- Creates a [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[async pipeline creation](https://gpuweb.github.io#async-pipeline-creation).
The returned [Promise](https://webidl.spec.whatwg.org/#idl-promise)
If pipeline creation fails, the returned [Promise](https://webidl.spec.whatwg.org/#idl-promise)[GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)[GPUError](https://gpuweb.github.io#gpuerror)
Note: Use of this method is preferred whenever possible, as it prevents blocking the
[queue timeline](https://gpuweb.github.io#queue-timeline) work on pipeline compilation.
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createRenderPipelineAsync(descriptor)](https://gpuweb.github.io#dom-gpudevice-createrenderpipelineasync) method.Parameter Type Nullable Optional Description `descriptor`
[GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)
✘ ✘ Description of the [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
**Returns:** [Promise](https://webidl.spec.whatwg.org/#idl-promise)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `contentTimeline` be the current [Content timeline](https://gpuweb.github.io#content-timeline).
  2. Let `promise` be [a new promise](https://webidl.spec.whatwg.org/#a-new-promise).
  3. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  4. Return `promise`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. Let `pipeline` be a new [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)`this`.[createRenderPipeline()](https://gpuweb.github.io#dom-gpudevice-createrenderpipeline)`descriptor`,
except capturing any errors as `error`, rather than dispatching them to the device.
  2. Let `event` occur upon the (successful or unsuccessful) completion of [pipeline creation](https://gpuweb.github.io/gpuweb/wgsl/#pipeline-creation) for `pipeline`.
  3. [Listen for timeline event](https://gpuweb.github.io#abstract-opdef-listen-for-timeline-event) `event` on `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[device timeline](https://gpuweb.github.io#device-timeline) of `this`.
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. If `pipeline` is [valid](https://gpuweb.github.io#abstract-opdef-valid) or `this` is [lost](https://gpuweb.github.io#abstract-opdef-invalid):
    1. Issue the following steps on `contentTimeline`:
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
      1. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise` with `pipeline`.
    2. Return.
Note: No errors are generated from a device which is lost.
See [§ 22 Errors & Debugging](https://gpuweb.github.io#errors-and-debugging).
  2. If `pipeline` is [invalid](https://gpuweb.github.io#abstract-opdef-invalid) and `error` is an [internal error](https://gpuweb.github.io#abstract-opdef-generate-an-internal-error),
issue the following steps on `contentTimeline`, and return.
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `promise` with a [GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)[reason](https://gpuweb.github.io#dom-gpupipelineerrorinit-reason)["internal"](https://gpuweb.github.io#dom-gpupipelineerrorreason-internal)
  3. If `pipeline` is [invalid](https://gpuweb.github.io#abstract-opdef-invalid) and `error` is a [validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error),
issue the following steps on `contentTimeline`, and return.
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `promise` with a [GPUPipelineError](https://gpuweb.github.io#gpupipelineerror)[reason](https://gpuweb.github.io#dom-gpupipelineerrorinit-reason)["validation"](https://gpuweb.github.io#dom-gpupipelineerrorreason-validation)

**Arguments:**

- [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)`descriptor`
- [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)`layout`
- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Return `true` if all of the following conditions are satisfied:
  - [validating GPUVertexState](https://gpuweb.github.io#abstract-opdef-validating-gpuvertexstate)(`device`, `descriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)`layout`) succeeds.
  - If `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)[provided](https://infra.spec.whatwg.org/#map-exists):
    - [validating GPUFragmentState](https://gpuweb.github.io#abstract-opdef-validating-gpufragmentstate)(`device`, `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)`layout`) succeeds.
    - If the [sample\_mask](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-sample_mask) builtin is a [shader stage output](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-output) of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)
      - `descriptor`.[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)[alphaToCoverageEnabled](https://gpuweb.github.io#dom-gpumultisamplestate-alphatocoverageenabled)`false`.
    - If the [frag\_depth](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-frag_depth) builtin is a [shader stage output](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-output) of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)
      - `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)[provided](https://infra.spec.whatwg.org/#map-exists), and `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)[format](https://gpuweb.github.io#dom-gpudepthstencilstate-format)[depth](https://gpuweb.github.io#aspect-depth) aspect.
    - 
           If`device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
      - The [sample\_mask](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-sample_mask) builtin must not be a [shader stage input](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-input) or [shader stage output](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-output) of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)
      - The [sample\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-sample_index) builtin must not be a [shader stage input](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-input) of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)
  - [validating GPUPrimitiveState](https://gpuweb.github.io#abstract-opdef-validating-gpuprimitivestate)(`descriptor`.[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)`device`) succeeds.
  - If `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)[provided](https://infra.spec.whatwg.org/#map-exists):
    - [validating GPUDepthStencilState](https://gpuweb.github.io#abstract-opdef-validating-gpudepthstencilstate)(`device`, `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)`descriptor`.[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[topology](https://gpuweb.github.io#dom-gpuprimitivestate-topology)
  - [validating GPUMultisampleState](https://gpuweb.github.io#abstract-opdef-validating-gpumultisamplestate)(`descriptor`.[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)
  - If `descriptor`.[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)[alphaToCoverageEnabled](https://gpuweb.github.io#dom-gpumultisamplestate-alphatocoverageenabled)
  - There must exist at least one attachment, either: 
    - A non-`null` value in `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)
    - A `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)
  - [validating inter-stage interfaces](https://gpuweb.github.io#abstract-opdef-validating-inter-stage-interfaces)(`device`, `descriptor`) returns `true`.

`variable`)

**Arguments:**

- shader binding declaration `variable`, a module-scope variable declaration reflected from a shader module

**Returns:** [boolean](https://webidl.spec.whatwg.org/#idl-boolean)

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. If the [interpolation](https://gpuweb.github.io/gpuweb/wgsl/#interpolation) of the `variable` is [linear](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-type-linear), return `false`.
2. If the [interpolation](https://gpuweb.github.io/gpuweb/wgsl/#interpolation) of the `variable` is [flat](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-type-flat) and the [interpolation sampling](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-sampling) is not [either](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-sampling-either), return `false`.
3. If the [interpolation sampling](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-sampling) of the `variable` is [sample](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-sampling-sample), return `false`.
4. Return 'true'

`device`,

`descriptor`)

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`
- [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)`descriptor`

**Returns:** [boolean](https://webidl.spec.whatwg.org/#idl-boolean)

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `maxVertexShaderOutputVariables` be `device`.limits.[maxInterStageShaderVariables](https://gpuweb.github.io#dom-supported-limits-maxinterstageshadervariables)
2. Let `maxVertexShaderOutputLocation` be `device`.limits.[maxInterStageShaderVariables](https://gpuweb.github.io#dom-supported-limits-maxinterstageshadervariables)
3. If `descriptor`.[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[topology](https://gpuweb.github.io#dom-gpuprimitivestate-topology)["point-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-point-list)
  1. Decrement `maxVertexShaderOutputVariables` by 1.
4. If [clip\_distances](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-clip_distances) is declared in the output of `descriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)
  1. Let `clipDistancesSize` be the array size of [clip\_distances](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-clip_distances).
  2. Decrement `maxVertexShaderOutputVariables` by ceil(`clipDistancesSize` / 4).
  3. Decrement `maxVertexShaderOutputLocation` by ceil(`clipDistancesSize` / 4).
5. Return `false` if any of the following requirements are unmet:
6. If `device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
  1. For each user-defined `output` of `descriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)
    1. If [validating Compatibility Mode shader binding](https://gpuweb.github.io#abstract-opdef-validating-compatibility-mode-shader-binding)(`output`) fails, return `false`.
7. If `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)[is provided](https://infra.spec.whatwg.org/#map-exists):
  1. Let `maxFragmentShaderInputVariables` be `device`.limits.[maxInterStageShaderVariables](https://gpuweb.github.io#dom-supported-limits-maxinterstageshadervariables)
  2. For each of the [Inter-Stage Builtins](https://gpuweb.github.io#inter-stage-builtins) that are an input of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)
    1. Decrement `maxFragmentShaderInputVariables` by 1.
  3. Return `false` if any of the following requirements are unmet:
    - For each user-defined input of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)`descriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)[location](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations), type, and [interpolation](https://gpuweb.github.io/gpuweb/wgsl/#interpolation) of the input.
Note: Vertex-only pipelines **can** have user-defined outputs in the vertex stage;
their values will be discarded.
    - There must be no more than `maxFragmentShaderInputVariables` user-defined inputs for `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)
  4. [Assert](https://infra.spec.whatwg.org/#assert) that the [location](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) of each user-defined input of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)`device`.limits.[maxInterStageShaderVariables](https://gpuweb.github.io#dom-supported-limits-maxinterstageshadervariables)
  5. If `device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
    1. For each user-defined `input` of `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)
      1. If [validating Compatibility Mode shader binding](https://gpuweb.github.io#abstract-opdef-validating-compatibility-mode-shader-binding)(`input`) fails, return `false`.
8. Return `true`.

The following [builtins](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values) are Inter-Stage Builtins, and count towards the
[maxInterStageShaderVariables](https://gpuweb.github.io#dom-supported-limits-maxinterstageshadervariables)

- `front_facing`
- `sample_index`
- `sample_mask`
- `primitive_index`
- `subgroup_invocation_id`
- `subgroup_size`

[Creating a simple](https://gpuweb.github.io#example-89112c19)

[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
const renderPipeline= gpuDevice. createRenderPipeline({ layout: pipelineLayout, vertex: { module: shaderModule, entryPoint: 'vertexMain' }, fragment: { module: shaderModule, entryPoint: 'fragmentMain' , targets: \[{ format: 'bgra8unorm' , }\], } }); 

#### 10.3.2. Primitive State

dictionary GPUPrimitiveState GPUPrimitiveTopology topology GPUIndexFormat stripIndexFormat GPUFrontFace frontFace GPUCullMode cullMode boolean unclippedDepth false ; };

[GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)

- `topology`,  of type [GPUPrimitiveTopology](https://gpuweb.github.io#enumdef-gpuprimitivetopology), defaulting to `"triangle-list"`
- The type of primitive to be constructed from the vertex inputs.
- `stripIndexFormat`,  of type [GPUIndexFormat](https://gpuweb.github.io#enumdef-gpuindexformat)
- For pipelines with strip topologies
(["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)["triangle-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-strip)["uint16"](https://gpuweb.github.io#dom-gpuindexformat-uint16)`0xFFFF` or ["uint32"](https://gpuweb.github.io#dom-gpuindexformat-uint32)`0xFFFFFFFF`).
It is not allowed on pipelines with non-strip topologies.
Note: Some implementations require knowledge of the primitive restart value to compile pipeline state objects. To use a strip-topology pipeline with an indexed draw call
([drawIndexed()](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindexed)[drawIndexedIndirect()](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindexedindirect)[setIndexBuffer()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setindexbuffer)
See [§ 23.2.3 Primitive Assembly](https://gpuweb.github.io#primitive-assembly) for additional details.
- `frontFace`,  of type [GPUFrontFace](https://gpuweb.github.io#enumdef-gpufrontface), defaulting to `"ccw"`
- Defines which polygons are considered [front-facing](https://gpuweb.github.io#front-facing).
- `cullMode`,  of type [GPUCullMode](https://gpuweb.github.io#enumdef-gpucullmode), defaulting to `"none"`
- Defines which polygon orientation will be culled, if any.
- `unclippedDepth`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- If true, indicates that [depth clipping](https://gpuweb.github.io#depth-clipping) is disabled.
Requires the ["depth-clip-control"](https://gpuweb.github.io#depth-clip-control)

`descriptor`,

`device`)

**Arguments:**

- [GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate)`descriptor`
- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Return `true` if all of the following conditions are satisfied:
  - If `descriptor`.[topology](https://gpuweb.github.io#dom-gpuprimitivestate-topology)["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)["triangle-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-strip)
    - `descriptor`.[stripIndexFormat](https://gpuweb.github.io#dom-gpuprimitivestate-stripindexformat)[provided](https://infra.spec.whatwg.org/#map-exists).
  - If `descriptor`.[unclippedDepth](https://gpuweb.github.io#dom-gpuprimitivestate-unclippeddepth)`true`:
    - ["depth-clip-control"](https://gpuweb.github.io#depth-clip-control)[enabled for](https://gpuweb.github.io#enabled-for) `device`.

enum GPUPrimitiveTopology "point-list" "line-list" "line-strip" "triangle-list" "triangle-strip" 

[GPUPrimitiveTopology](https://gpuweb.github.io#enumdef-gpuprimitivetopology)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[§ 23.2.5 Rasterization](https://gpuweb.github.io#rasterization) for additional details:

- `"point-list"`
- Each vertex defines a point primitive.
- `"line-list"`
- Each consecutive pair of two vertices defines a line primitive.
- `"line-strip"`
- Each vertex after the first defines a line primitive between it and the previous vertex.
- `"triangle-list"`
- Each consecutive triplet of three vertices defines a triangle primitive.
- `"triangle-strip"`
- Each vertex after the first two defines a triangle primitive between it and the previous two vertices.

enum GPUFrontFace "ccw" "cw" 

[GPUFrontFace](https://gpuweb.github.io#enumdef-gpufrontface)[front-facing](https://gpuweb.github.io#front-facing) by a [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[§ 23.2.5.4 Polygon Rasterization](https://gpuweb.github.io#polygon-rasterization) for additional details:

- `"ccw"`
- Polygons with vertices whose framebuffer coordinates are given in counter-clockwise order
are considered [front-facing](https://gpuweb.github.io#front-facing).
- `"cw"`
- Polygons with vertices whose framebuffer coordinates are given in clockwise order are
considered [front-facing](https://gpuweb.github.io#front-facing).

enum GPUCullMode "none" "front" "back" 

[GPUPrimitiveTopology](https://gpuweb.github.io#enumdef-gpuprimitivetopology)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[§ 23.2.5.4 Polygon Rasterization](https://gpuweb.github.io#polygon-rasterization) for additional details:

- `"none"`
- No polygons are discarded.
- `"front"`
- [Front-facing](https://gpuweb.github.io#front-facing) polygons are discarded.
- `"back"`
- [Back-facing](https://gpuweb.github.io#back-facing) polygons are discarded.

Note: [GPUFrontFace](https://gpuweb.github.io#enumdef-gpufrontface)[GPUCullMode](https://gpuweb.github.io#enumdef-gpucullmode)["point-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-point-list)["line-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-list)["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)

#### 10.3.3. Multisample State

dictionary GPUMultisampleState GPUSize32 count GPUSampleMask mask boolean alphaToCoverageEnabled false ; };

[GPUMultisampleState](https://gpuweb.github.io#dictdef-gpumultisamplestate)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)

- `count`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32), defaulting to `1`
- Number of samples per pixel. This [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)[sampleCount](https://gpuweb.github.io#dom-gputexturedescriptor-samplecount)
- `mask`,  of type [GPUSampleMask](https://gpuweb.github.io#typedefdef-gpusamplemask), defaulting to `0xFFFFFFFF`
- Mask determining which samples are written to.
- `alphaToCoverageEnabled`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- When `true` indicates that a fragment’s alpha channel should be used to generate a sample
coverage mask.

`descriptor`)

**Arguments:**

- [GPUMultisampleState](https://gpuweb.github.io#dictdef-gpumultisamplestate)`descriptor`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Return `true` if all of the following conditions are satisfied:
  - `descriptor`.[count](https://gpuweb.github.io#dom-gpumultisamplestate-count)
  - If `descriptor`.[alphaToCoverageEnabled](https://gpuweb.github.io#dom-gpumultisamplestate-alphatocoverageenabled)`true`:
    - `descriptor`.[count](https://gpuweb.github.io#dom-gpumultisamplestate-count)

#### 10.3.4. Fragment State

dictionary GPUFragmentState GPUProgrammableStage required sequence GPUColorTargetState targets 

- `targets`,  of type `sequence<GPUColorTargetState?>`
- A list of [GPUColorTargetState](https://gpuweb.github.io#dictdef-gpucolortargetstate)

`device`,

`descriptor`,

`layout`)

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`
- [GPUFragmentState](https://gpuweb.github.io#dictdef-gpufragmentstate)`descriptor`
- [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)`layout`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Return `true` if all of the following requirements are met:
  - [validating GPUProgrammableStage](https://gpuweb.github.io#abstract-opdef-validating-gpuprogrammablestage)([FRAGMENT](https://gpuweb.github.io#dom-gpushaderstage-fragment)`descriptor`, `layout`, `device`) succeeds.
  - `descriptor`.[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)[size](https://infra.spec.whatwg.org/#list-size) must be ≤ `device`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxColorAttachments](https://gpuweb.github.io#dom-supported-limits-maxcolorattachments)
  - For each [shader stage output](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-output) `output`:
    - `output`’s [location](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) must be \< `device`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxColorAttachments](https://gpuweb.github.io#dom-supported-limits-maxcolorattachments)
  - Let `entryPoint` be [get the entry point](https://gpuweb.github.io#abstract-opdef-get-the-entry-point)([FRAGMENT](https://gpuweb.github.io#dom-gpushaderstage-fragment)`descriptor`).
  - Let `usesDualSourceBlending` be `false`.
  - [For each](https://infra.spec.whatwg.org/#list-iterate) `index` of the [indices](https://infra.spec.whatwg.org/#list-get-the-indices) of `descriptor`.[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)`null` value `colorState`:
    - `colorState`.[format](https://gpuweb.github.io#dom-gpucolortargetstate-format)[§ 26.1.1 Plain color formats](https://gpuweb.github.io#plain-color-formats) with [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)
    - `colorState`.[writeMask](https://gpuweb.github.io#dom-gpucolortargetstate-writemask)
    - If `colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[provided](https://infra.spec.whatwg.org/#map-exists):
      - The `colorState`.[format](https://gpuweb.github.io#dom-gpucolortargetstate-format)[blendable](https://gpuweb.github.io#blendable).
      - `colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[color](https://gpuweb.github.io#dom-gpublendstate-color)[valid GPUBlendComponent](https://gpuweb.github.io#valid-gpublendcomponent).
      - `colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[alpha](https://gpuweb.github.io#dom-gpublendstate-alpha)[valid GPUBlendComponent](https://gpuweb.github.io#valid-gpublendcomponent).
      - If `colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[color](https://gpuweb.github.io#dom-gpublendstate-color)[srcFactor](https://gpuweb.github.io#dom-gpublendcomponent-srcfactor)`colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[color](https://gpuweb.github.io#dom-gpublendstate-color)[dstFactor](https://gpuweb.github.io#dom-gpublendcomponent-dstfactor)`colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[alpha](https://gpuweb.github.io#dom-gpublendstate-alpha)[srcFactor](https://gpuweb.github.io#dom-gpublendcomponent-srcfactor)`colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[alpha](https://gpuweb.github.io#dom-gpublendstate-alpha)[dstFactor](https://gpuweb.github.io#dom-gpublendcomponent-dstfactor)["src1"](https://gpuweb.github.io#dom-gpublendfactor-src1)["one-minus-src1"](https://gpuweb.github.io#dom-gpublendfactor-one-minus-src1)["src1-alpha"](https://gpuweb.github.io#dom-gpublendfactor-src1-alpha)["one-minus-src1-alpha"](https://gpuweb.github.io#dom-gpublendfactor-one-minus-src1-alpha)
        - Set `usesDualSourceBlending` to `true`.
    - For each [shader stage output](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-output) value `output` with [location](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) attribute equal to `index` in `entryPoint`:
      - For each component in `colorState`.[format](https://gpuweb.github.io#dom-gpucolortargetstate-format)`output`.
(That is, RGBA requires vec4, RGB requires vec3 or vec4, RG requires vec2 or vec3 or vec4.)
      - If the [GPUTextureSampleType](https://gpuweb.github.io#enumdef-gputexturesampletype)`colorState`.[format](https://gpuweb.github.io#dom-gpucolortargetstate-format)[§ 26.1 Texture Format Capabilities](https://gpuweb.github.io#texture-format-caps)) are:
        - ["float"](https://gpuweb.github.io#dom-gputexturesampletype-float)
["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float)
        - `output` must have a floating-point scalar type.
        - ["sint"](https://gpuweb.github.io#dom-gputexturesampletype-sint)
        - `output` must have a signed integer scalar type.
        - ["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint)
        - `output` must have an unsigned integer scalar type.
      - If `colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[provided](https://infra.spec.whatwg.org/#map-exists) and `colorState`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[color](https://gpuweb.github.io#dom-gpublendstate-color)[srcFactor](https://gpuweb.github.io#dom-gpublendcomponent-srcfactor)[dstFactor](https://gpuweb.github.io#dom-gpublendcomponent-dstfactor)["src-alpha"](https://gpuweb.github.io#dom-gpublendfactor-src-alpha)["one-minus-src-alpha"](https://gpuweb.github.io#dom-gpublendfactor-one-minus-src-alpha)["src-alpha-saturated"](https://gpuweb.github.io#dom-gpublendfactor-src-alpha-saturated)["src1-alpha"](https://gpuweb.github.io#dom-gpublendfactor-src1-alpha)["one-minus-src1-alpha"](https://gpuweb.github.io#dom-gpublendfactor-one-minus-src1-alpha)
        - `output` must have an alpha channel (that is, it must be a vec4).
    - If `colorState`.[writeMask](https://gpuweb.github.io#dom-gpucolortargetstate-writemask)
      - `entryPoint` must have a [shader stage output](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-output) with [location](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) equal to `index` and [blend\_src](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) omitted or equal to 0.
  - If `usesDualSourceBlending` is `true`:
    - `descriptor`.[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)[size](https://infra.spec.whatwg.org/#list-size) must be 1.
    - All the [shader stage outputs](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-output) with [location](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) in `entryPoint` must be in one
struct and [use dual source blending](https://gpuweb.github.io/gpuweb/wgsl/#use-dual-source-blending).
  - [Validating GPUFragmentState’s color attachment bytes per sample](https://gpuweb.github.io#abstract-opdef-validating-gpufragmentstates-color-attachment-bytes-per-sample)(`device`, `descriptor`.[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)
  - 
         If`device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
    - All non-null [GPUColorTargetState](https://gpuweb.github.io#dictdef-gpucolortargetstate)`colorState` in `descriptor`.[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)
    - For each `function` in the [functions in the shader stage](https://gpuweb.github.io/gpuweb/wgsl/#functions-in-a-shader-stage) rooted at `entryPoint`:
      - `function` must not be [dpdxFine](https://gpuweb.github.io/gpuweb/wgsl/#dpdxFine-builtin), [dpdyFine](https://gpuweb.github.io/gpuweb/wgsl/#dpdyFine-builtin), or [fwidthFine](https://gpuweb.github.io/gpuweb/wgsl/#fwidthFine-builtin).

`device`,

`targets`)

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`
- [sequence](https://webidl.spec.whatwg.org/#idl-sequence)\<[GPUColorTargetState](https://gpuweb.github.io#dictdef-gpucolortargetstate)`targets`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `formats` be an empty [list](https://infra.spec.whatwg.org/#list)\<[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
2. For each `target` in `targets`:
3. [Calculating color attachment bytes per sample](https://gpuweb.github.io#abstract-opdef-calculating-color-attachment-bytes-per-sample)(`formats`) must be ≤ `device`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxColorAttachmentBytesPerSample](https://gpuweb.github.io#dom-supported-limits-maxcolorattachmentbytespersample)

Note: The fragment shader may output more values than what the pipeline uses. If that is the case the values are ignored.

[GPUBlendComponent](https://gpuweb.github.io#dictdef-gpublendcomponent)
`component`is a valid GPUBlendComponent with logical

[device](https://gpuweb.github.io#device)

`device`if it meets

the following requirements:

#### 10.3.5. Color Target State

dictionary GPUColorTargetState required GPUTextureFormat format GPUBlendState blend GPUColorWriteFlags writeMask 

- `format`,  of type [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- The [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[GPUTextureView](https://gpuweb.github.io#gputextureview)
- `blend`,  of type [GPUBlendState](https://gpuweb.github.io#dictdef-gpublendstate)
- The blending behavior for this color target. If left undefined, disables blending for this color target.
- `writeMask`,  of type [GPUColorWriteFlags](https://gpuweb.github.io#typedefdef-gpucolorwriteflags), defaulting to `0xF`
- Bitmask controlling which channels are are written to when drawing to this color target.

dictionary GPUBlendState required GPUBlendComponent color required GPUBlendComponent alpha 

- `color`,  of type [GPUBlendComponent](https://gpuweb.github.io#dictdef-gpublendcomponent)
- Defines the blending behavior of the corresponding render target for color channels.
- `alpha`,  of type [GPUBlendComponent](https://gpuweb.github.io#dictdef-gpublendcomponent)
- Defines the blending behavior of the corresponding render target for the alpha channel.

typedef \[EnforceRange unsigned long GPUColorWriteFlags Exposed Window ,Worker ),SecureContext namespace GPUColorWrite const GPUFlagsConstant RED const GPUFlagsConstant GREEN const GPUFlagsConstant BLUE const GPUFlagsConstant ALPHA const GPUFlagsConstant ALL 

##### 10.3.5.1. Blend State

dictionary GPUBlendComponent GPUBlendOperation operation GPUBlendFactor srcFactor GPUBlendFactor dstFactor 

[GPUBlendComponent](https://gpuweb.github.io#dictdef-gpublendcomponent)

- `operation`,  of type [GPUBlendOperation](https://gpuweb.github.io#enumdef-gpublendoperation), defaulting to `"add"`
- Defines the [GPUBlendOperation](https://gpuweb.github.io#enumdef-gpublendoperation)
- `srcFactor`,  of type [GPUBlendFactor](https://gpuweb.github.io#enumdef-gpublendfactor), defaulting to `"one"`
- Defines the [GPUBlendFactor](https://gpuweb.github.io#enumdef-gpublendfactor)
- `dstFactor`,  of type [GPUBlendFactor](https://gpuweb.github.io#enumdef-gpublendfactor), defaulting to `"zero"`
- Defines the [GPUBlendFactor](https://gpuweb.github.io#enumdef-gpublendfactor)

The following tables use this notation to describe color components for a given fragment location:

| **`RGBA`** | Color output by the fragment shader for the color attachment. If the shader doesn’t return an alpha channel, src-alpha blend factors cannot be used. | 
| **`RGBA`** | Color output by the fragment shader for the color attachment with ["@blend\_src" attribute](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) equal to `1`.                 If the shader doesn’t return an alpha channel, src1-alpha blend factors cannot be used. | 
| **`RGBA`** | Color currently in the color attachment.                 Missing green/blue/alpha channels default to `0, 0, 1`, respectively. | 
| **`RGBA`** | The current [\[\[blendConstant\]\]](https://gpuweb.github.io#dom-renderstate-blendconstant-slot). | 
| **`RGBA`** | The source blend factor components, as defined by [srcFactor](https://gpuweb.github.io#dom-gpublendcomponent-srcfactor). | 
| **`RGBA`** | The destination blend factor components, as defined by [dstFactor](https://gpuweb.github.io#dom-gpublendcomponent-dstfactor). | 

enum GPUBlendFactor "zero" "one" "src" "one-minus-src" "src-alpha" "one-minus-src-alpha" "dst" "one-minus-dst" "dst-alpha" "one-minus-dst-alpha" "src-alpha-saturated" "constant" "one-minus-constant" "src1" "one-minus-src1" "src1-alpha" "one-minus-src1-alpha" 

[GPUBlendFactor](https://gpuweb.github.io#enumdef-gpublendfactor)

| GPUBlendFactor | Blend factor RGBA components | [Feature](https://gpuweb.github.io#feature) | 
|---|---|---|
| `"zero"` | `(0, 0, 0, 0)` |  | 
| `"one"` | `(1, 1, 1, 1)` |  | 
| `"src"` | `(R`<sub>src</sub>, G<sub>src</sub>, B<sub>src</sub>, A<sub>src</sub>) |  | 
| `"one-minus-src"` | `(1 - R`<sub>src</sub>, 1 - G<sub>src</sub>, 1 - B<sub>src</sub>, 1 - A<sub>src</sub>) |  | 
| `"src-alpha"` | `(A`<sub>src</sub>, A<sub>src</sub>, A<sub>src</sub>, A<sub>src</sub>) |  | 
| `"one-minus-src-alpha"` | `(1 - A`<sub>src</sub>, 1 - A<sub>src</sub>, 1 - A<sub>src</sub>, 1 - A<sub>src</sub>) |  | 
| `"dst"` | `(R`<sub>dst</sub>, G<sub>dst</sub>, B<sub>dst</sub>, A<sub>dst</sub>) |  | 
| `"one-minus-dst"` | `(1 - R`<sub>dst</sub>, 1 - G<sub>dst</sub>, 1 - B<sub>dst</sub>, 1 - A<sub>dst</sub>) |  | 
| `"dst-alpha"` | `(A`<sub>dst</sub>, A<sub>dst</sub>, A<sub>dst</sub>, A<sub>dst</sub>) |  | 
| `"one-minus-dst-alpha"` | `(1 - A`<sub>dst</sub>, 1 - A<sub>dst</sub>, 1 - A<sub>dst</sub>, 1 - A<sub>dst</sub>) |  | 
| `"src-alpha-saturated"` | `(min(A`<sub>src</sub>, 1 - A<sub>dst</sub>), min(A<sub>src</sub>, 1 - A<sub>dst</sub>), min(A<sub>src</sub>, 1 - A<sub>dst</sub>), 1) |  | 
| `"constant"` | `(R`<sub>const</sub>, G<sub>const</sub>, B<sub>const</sub>, A<sub>const</sub>) |  | 
| `"one-minus-constant"` | `(1 - R`<sub>const</sub>, 1 - G<sub>const</sub>, 1 - B<sub>const</sub>, 1 - A<sub>const</sub>) |  | 
| `"src1"` | `(R`<sub>src1</sub>, G<sub>src1</sub>, B<sub>src1</sub>, A<sub>src1</sub>) | [dual-source-blending](https://gpuweb.github.io#dom-gpufeaturename-dual-source-blending) | 
| `"one-minus-src1"` | `(1 - R`<sub>src1</sub>, 1 - G<sub>src1</sub>, 1 - B<sub>src1</sub>, 1 - A<sub>src1</sub>) |  | 
| `"src1-alpha"` | `(A`<sub>src1</sub>, A<sub>src1</sub>, A<sub>src1</sub>, A<sub>src1</sub>) |  | 
| `"one-minus-src1-alpha"` | `(1 - A`<sub>src1</sub>, 1 - A<sub>src1</sub>, 1 - A<sub>src1</sub>, 1 - A<sub>src1</sub>) |  | 

enum GPUBlendOperation "add" "subtract" "reverse-subtract" "min" "max" 

[GPUBlendOperation](https://gpuweb.github.io#enumdef-gpublendoperation)

| GPUBlendOperation | RGBA Components | 
|---|---|
| `"add"` | `RGBA`<sub>src</sub> × RGBA<sub>srcFactor</sub> + RGBA<sub>dst</sub> × RGBA<sub>dstFactor</sub> | 
| `"subtract"` | `RGBA`<sub>src</sub> × RGBA<sub>srcFactor</sub> - RGBA<sub>dst</sub> × RGBA<sub>dstFactor</sub> | 
| `"reverse-subtract"` | `RGBA`<sub>dst</sub> × RGBA<sub>dstFactor</sub> - RGBA<sub>src</sub> × RGBA<sub>srcFactor</sub> | 
| `"min"` | `min(RGBA`<sub>src</sub>, RGBA<sub>dst</sub>) | 
| `"max"` | `max(RGBA`<sub>src</sub>, RGBA<sub>dst</sub>) | 

#### 10.3.6. Depth/Stencil State

dictionary GPUDepthStencilState required GPUTextureFormat format boolean depthWriteEnabled GPUCompareFunction depthCompare GPUStencilFaceState stencilFront GPUStencilFaceState stencilBack GPUStencilValue stencilReadMask GPUStencilValue stencilWriteMask GPUDepthBias depthBias float depthBiasSlopeScale float depthBiasClamp 

[GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)

- `format`,  of type [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- The [format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
- `depthWriteEnabled`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- Indicates if this [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
- `depthCompare`,  of type [GPUCompareFunction](https://gpuweb.github.io#enumdef-gpucomparefunction)
- The comparison operation used to test fragment depths against
[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
- `stencilFront`,  of type [GPUStencilFaceState](https://gpuweb.github.io#dictdef-gpustencilfacestate), defaulting to `{}`
- Defines how stencil comparisons and operations are performed for front-facing primitives.
- `stencilBack`,  of type [GPUStencilFaceState](https://gpuweb.github.io#dictdef-gpustencilfacestate), defaulting to `{}`
- Defines how stencil comparisons and operations are performed for back-facing primitives.
- `stencilReadMask`,  of type [GPUStencilValue](https://gpuweb.github.io#typedefdef-gpustencilvalue), defaulting to `0xFFFFFFFF`
- Bitmask controlling which [depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
- `stencilWriteMask`,  of type [GPUStencilValue](https://gpuweb.github.io#typedefdef-gpustencilvalue), defaulting to `0xFFFFFFFF`
- Bitmask controlling which [depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
- `depthBias`,  of type [GPUDepthBias](https://gpuweb.github.io#typedefdef-gpudepthbias), defaulting to `0`
- Constant depth bias added to each triangle fragment. See [biased fragment depth](https://gpuweb.github.io#abstract-opdef-biased-fragment-depth) for details.
- `depthBiasSlopeScale`,  of type [float](https://webidl.spec.whatwg.org/#idl-float), defaulting to `0`
- Depth bias that scales with the triangle fragment’s slope. See [biased fragment depth](https://gpuweb.github.io#abstract-opdef-biased-fragment-depth) for details.
- `depthBiasClamp`,  of type [float](https://webidl.spec.whatwg.org/#idl-float), defaulting to `0`
- The maximum depth bias of a triangle fragment. See [biased fragment depth](https://gpuweb.github.io#abstract-opdef-biased-fragment-depth) for details.

Note: [depthBias](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbias)[depthBiasSlopeScale](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasslopescale)[depthBiasClamp](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasclamp)["point-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-point-list)["line-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-list)["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)

[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
`attachment`when drawing using

[GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)
`state`is calculated by running the following

[queue timeline](https://gpuweb.github.io#queue-timeline)steps:

1. Let `format` be `attachment`.[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)
2. Let `r` be the minimum positive representable value > `0` in the `format` converted to a 32-bit float.
3. Let `maxDepthSlope` be the maximum of the horizontal and vertical slopes of the fragment’s depth value.
4. If `format` is a **unorm** format:
  1. Let `bias` be `(float)`.`state`.[depthBias](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbias)`r` + `state`.[depthBiasSlopeScale](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasslopescale)`maxDepthSlope`
Otherwise, if `format` is a **float** format:
  1. Let `bias` be `(float)`.`state`.[depthBias](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbias)`r`) + `state`.[depthBiasSlopeScale](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasslopescale)`maxDepthSlope`
5. If `state`.[depthBiasClamp](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasclamp)`0`:
  1. Set `bias` to `min(`.`state`.[depthBiasClamp](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasclamp)`bias`)
Otherwise, if `state`.[depthBiasClamp](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasclamp)`0`:
  1. Set `bias` to `max(`.`state`.[depthBiasClamp](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasclamp)`bias`)
6. If `state`.[depthBias](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbias)`0` or `state`.[depthBiasSlopeScale](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasslopescale)`0`:
  1. Set the fragment depth value to `fragment depth value +` `bias`

`device`,

`descriptor`,

`topology`)

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`
- [GPUDepthStencilState](https://gpuweb.github.io#dictdef-gpudepthstencilstate)`descriptor`
- [GPUPrimitiveTopology](https://gpuweb.github.io#enumdef-gpuprimitivetopology)`topology`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Return `true` if, and only if, all of the following conditions are satisfied:
  - `descriptor`.[format](https://gpuweb.github.io#dom-gpudepthstencilstate-format)[depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format).
  - If `descriptor`.[depthWriteEnabled](https://gpuweb.github.io#dom-gpudepthstencilstate-depthwriteenabled)`true` or `descriptor`.[depthCompare](https://gpuweb.github.io#dom-gpudepthstencilstate-depthcompare)[provided](https://infra.spec.whatwg.org/#map-exists) and not ["always"](https://gpuweb.github.io#dom-gpucomparefunction-always)
    - `descriptor`.[format](https://gpuweb.github.io#dom-gpudepthstencilstate-format)
  - If `descriptor`.[stencilFront](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilfront)`descriptor`.[stencilBack](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilback)
    - `descriptor`.[format](https://gpuweb.github.io#dom-gpudepthstencilstate-format)
  - If `descriptor`.[format](https://gpuweb.github.io#dom-gpudepthstencilstate-format)
    - `descriptor`.[depthWriteEnabled](https://gpuweb.github.io#dom-gpudepthstencilstate-depthwriteenabled)[provided](https://infra.spec.whatwg.org/#map-exists).
    - `descriptor`.[depthCompare](https://gpuweb.github.io#dom-gpudepthstencilstate-depthcompare)[provided](https://infra.spec.whatwg.org/#map-exists) if:
      - `descriptor`.[depthWriteEnabled](https://gpuweb.github.io#dom-gpudepthstencilstate-depthwriteenabled)`true`, or
      - `descriptor`.[stencilFront](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilfront)[depthFailOp](https://gpuweb.github.io#dom-gpustencilfacestate-depthfailop)["keep"](https://gpuweb.github.io#dom-gpustenciloperation-keep)
      - `descriptor`.[stencilBack](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilback)[depthFailOp](https://gpuweb.github.io#dom-gpustencilfacestate-depthfailop)["keep"](https://gpuweb.github.io#dom-gpustenciloperation-keep)
  - If `topology` is ["point-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-point-list)["line-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-list)["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)
    - `descriptor`.[depthBias](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbias)
    - `descriptor`.[depthBiasSlopeScale](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasslopescale)
    - `descriptor`.[depthBiasClamp](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasclamp)
  - 
         If`device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
    - `descriptor`.[depthBiasClamp](https://gpuweb.github.io#dom-gpudepthstencilstate-depthbiasclamp)

dictionary GPUStencilFaceState GPUCompareFunction compare GPUStencilOperation failOp GPUStencilOperation depthFailOp GPUStencilOperation passOp 

[GPUStencilFaceState](https://gpuweb.github.io#dictdef-gpustencilfacestate)

- `compare`,  of type [GPUCompareFunction](https://gpuweb.github.io#enumdef-gpucomparefunction), defaulting to `"always"`
- The [GPUCompareFunction](https://gpuweb.github.io#enumdef-gpucomparefunction)[\[\[stencilReference\]\]](https://gpuweb.github.io#dom-renderstate-stencilreference-slot)[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
- `failOp`,  of type [GPUStencilOperation](https://gpuweb.github.io#enumdef-gpustenciloperation), defaulting to `"keep"`
- The [GPUStencilOperation](https://gpuweb.github.io#enumdef-gpustenciloperation)[compare](https://gpuweb.github.io#dom-gpustencilfacestate-compare)
- `depthFailOp`,  of type [GPUStencilOperation](https://gpuweb.github.io#enumdef-gpustenciloperation), defaulting to `"keep"`
- The [GPUStencilOperation](https://gpuweb.github.io#enumdef-gpustenciloperation)[depthCompare](https://gpuweb.github.io#dom-gpudepthstencilstate-depthcompare)
- `passOp`,  of type [GPUStencilOperation](https://gpuweb.github.io#enumdef-gpustenciloperation), defaulting to `"keep"`
- The [GPUStencilOperation](https://gpuweb.github.io#enumdef-gpustenciloperation)[compare](https://gpuweb.github.io#dom-gpustencilfacestate-compare)

enum GPUStencilOperation "keep" "zero" "replace" "invert" "increment-clamp" "decrement-clamp" "increment-wrap" "decrement-wrap" 

[GPUStencilOperation](https://gpuweb.github.io#enumdef-gpustenciloperation)

- `"keep"`
- Keep the current stencil value.
- `"zero"`
- Set the stencil value to `0`.
- `"replace"`
- Set the stencil value to [\[\[stencilReference\]\]](https://gpuweb.github.io#dom-renderstate-stencilreference-slot)
- `"invert"`
- Bitwise-invert the current stencil value.
- `"increment-clamp"`
- Increments the current stencil value, clamping to the maximum representable value of the
[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
- `"decrement-clamp"`
- Decrement the current stencil value, clamping to `0`.
- `"increment-wrap"`
- Increments the current stencil value, wrapping to zero if the value exceeds the maximum
representable value of the [depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
- `"decrement-wrap"`
- Decrement the current stencil value, wrapping to the maximum representable value of the
[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)`0`.

#### 10.3.7. Vertex State

enum GPUIndexFormat "uint16" "uint32" 

The index format determines both the data type of index values in a buffer and, when used with
strip primitive topologies (["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)["triangle-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-strip)

[GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate)[stripIndexFormat](https://gpuweb.github.io#dom-gpuprimitivestate-stripindexformat)[primitive restart value](https://gpuweb.github.io#primitive-restart-value) that will be used is known at pipeline
creation time. [GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate)[setIndexBuffer()](https://gpuweb.github.io#dom-gpurendercommandsmixin-setindexbuffer)

| Index format | Byte size | Primitive restart value | 
|---|---|---|
| `"uint16"` | 2 | 0xFFFF | 
| `"uint32"` | 4 | 0xFFFFFFFF | 

##### 10.3.7.1. Vertex Formats

The [GPUVertexFormat](https://gpuweb.github.io#enumdef-gpuvertexformat)[vertex data type](https://gpuweb.github.io#vertex-data-type) for the component.

Each vertex data type can map to any [WGSL scalar type](https://gpuweb.github.io/gpuweb/wgsl/#scalar-types) of the same base type,
regardless of the bits per component:

| Vertex format prefix | Vertex data type | Compatible WGSL types | 
|---|---|---|
| `uint` | unsigned int | `u32` | 
| `sint` | signed int | `i32` | 
| `unorm` | unsigned normalized | `f16`, `f32` | 
| `snorm` | signed normalized |  | 
| `float` | floating point |  | 

The multi-component formats specify the number of components after "x". Mismatches in the number of components between the vertex format and shader type are allowed, with components being either dropped or filled with default values to compensate.

[A vertex attribute with a format of](https://gpuweb.github.io#example-f3171d21)

["unorm8x2"](https://gpuweb.github.io#dom-gpuvertexformat-unorm8x2)`[0x7F, 0xFF]`
    can be accessed in the shader with the following types:
    
    | Shader type | Shader value | 
|---|---|
| `f16` | `0.5h` | 
| `f32` | `0.5f` | 
| `vec2<f16>` | `vec2(0.5h, 1.0h)` | 
| `vec2<f32>` | `vec2(0.5f, 1.0f)` | 
| `vec3<f16>` | `vec2(0.5h, 1.0h, 0.0h)` | 
| `vec3<f32>` | `vec2(0.5f, 1.0f, 0.0f)` | 
| `vec4<f16>` | `vec2(0.5h, 1.0h, 0.0h, 1.0h)` | 
| `vec4<f32>` | `vec2(0.5f, 1.0f, 0.0f, 1.0f)` | 

See [§ 23.2.2 Vertex Processing](https://gpuweb.github.io#vertex-processing) for additional information about how vertex formats are exposed in the
shader.

enum GPUVertexFormat "uint8" "uint8x2" "uint8x4" "sint8" "sint8x2" "sint8x4" "unorm8" "unorm8x2" "unorm8x4" "snorm8" "snorm8x2" "snorm8x4" "uint16" "uint16x2" "uint16x4" "sint16" "sint16x2" "sint16x4" "unorm16" "unorm16x2" "unorm16x4" "snorm16" "snorm16x2" "snorm16x4" "float16" "float16x2" "float16x4" "float32" "float32x2" "float32x3" "float32x4" "uint32" "uint32x2" "uint32x3" "uint32x4" "sint32" "sint32x2" "sint32x3" "sint32x4" "unorm10-10-10-2" "unorm8x4-bgra" "snorm10-10-10-2" 

| Vertex format | Data type | Components | byteSize | Example WGSL type | 
|---|---|---|---|---|
| `"uint8"` | unsigned int | 1 | 1 | `u32` | 
| `"uint8x2"` | unsigned int | 2 | 2 | `vec2<u32>` | 
| `"uint8x4"` | unsigned int | 4 | 4 | `vec4<u32>` | 
| `"sint8"` | signed int | 1 | 1 | `i32` | 
| `"sint8x2"` | signed int | 2 | 2 | `vec2<i32>` | 
| `"sint8x4"` | signed int | 4 | 4 | `vec4<i32>` | 
| `"unorm8"` | unsigned normalized | 1 | 1 | `f32` | 
| `"unorm8x2"` | unsigned normalized | 2 | 2 | `vec2<f32>` | 
| `"unorm8x4"` | unsigned normalized | 4 | 4 | `vec4<f32>` | 
| `"snorm8"` | signed normalized | 1 | 1 | `f32` | 
| `"snorm8x2"` | signed normalized | 2 | 2 | `vec2<f32>` | 
| `"snorm8x4"` | signed normalized | 4 | 4 | `vec4<f32>` | 
| `"uint16"` | unsigned int | 1 | 2 | `u32` | 
| `"uint16x2"` | unsigned int | 2 | 4 | `vec2<u32>` | 
| `"uint16x4"` | unsigned int | 4 | 8 | `vec4<u32>` | 
| `"sint16"` | signed int | 1 | 2 | `i32` | 
| `"sint16x2"` | signed int | 2 | 4 | `vec2<i32>` | 
| `"sint16x4"` | signed int | 4 | 8 | `vec4<i32>` | 
| `"unorm16"` | unsigned normalized | 1 | 2 | `f32` | 
| `"unorm16x2"` | unsigned normalized | 2 | 4 | `vec2<f32>` | 
| `"unorm16x4"` | unsigned normalized | 4 | 8 | `vec4<f32>` | 
| `"snorm16"` | signed normalized | 1 | 2 | `f32` | 
| `"snorm16x2"` | signed normalized | 2 | 4 | `vec2<f32>` | 
| `"snorm16x4"` | signed normalized | 4 | 8 | `vec4<f32>` | 
| `"float16"` | float | 1 | 2 | `f32` | 
| `"float16x2"` | float | 2 | 4 | `vec2<f16>` | 
| `"float16x4"` | float | 4 | 8 | `vec4<f16>` | 
| `"float32"` | float | 1 | 4 | `f32` | 
| `"float32x2"` | float | 2 | 8 | `vec2<f32>` | 
| `"float32x3"` | float | 3 | 12 | `vec3<f32>` | 
| `"float32x4"` | float | 4 | 16 | `vec4<f32>` | 
| `"uint32"` | unsigned int | 1 | 4 | `u32` | 
| `"uint32x2"` | unsigned int | 2 | 8 | `vec2<u32>` | 
| `"uint32x3"` | unsigned int | 3 | 12 | `vec3<u32>` | 
| `"uint32x4"` | unsigned int | 4 | 16 | `vec4<u32>` | 
| `"sint32"` | signed int | 1 | 4 | `i32` | 
| `"sint32x2"` | signed int | 2 | 8 | `vec2<i32>` | 
| `"sint32x3"` | signed int | 3 | 12 | `vec3<i32>` | 
| `"sint32x4"` | signed int | 4 | 16 | `vec4<i32>` | 
| `"unorm10-10-10-2"` | unsigned normalized | 4 | 4 | `vec4<f32>` | 
| `"unorm8x4-bgra"` | unsigned normalized | 4 | 4 | `vec4<f32>` | 
| `"snorm10-10-10-2"` | signed normalized | 4 | 4 | `vec4<f32>` | 

enum GPUVertexStepMode "vertex" "instance" 

The step mode configures how an address for vertex buffer data is computed, based on the current vertex or instance index:

- `"vertex"`
- The address is advanced by [arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)
- `"instance"`
- The address is advanced by [arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)

dictionary GPUVertexState GPUProgrammableStage sequence GPUVertexBufferLayout buffers 

- `buffers`,  of type `sequence<GPUVertexBufferLayout?>`, defaulting to `[]`
- A list of [GPUVertexBufferLayout](https://gpuweb.github.io#dictdef-gpuvertexbufferlayout)

A vertex buffer is, conceptually, a view into buffer memory as an *array of structures*.
[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)*elements* of that array.
Each element of a vertex buffer is like a *structure* with a memory layout defined by its
[attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes)*members* of the structure.

Each [GPUVertexAttribute](https://gpuweb.github.io#dictdef-gpuvertexattribute)[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)[offset](https://gpuweb.github.io#dom-gpuvertexattribute-offset)

Each attribute appears as a separate input in a vertex shader, each bound by a numeric *location*,
which is specified by [shaderLocation](https://gpuweb.github.io#dom-gpuvertexattribute-shaderlocation)[GPUVertexState](https://gpuweb.github.io#dictdef-gpuvertexstate)

dictionary GPUVertexBufferLayout required GPUSize64 arrayStride GPUVertexStepMode stepMode required sequence GPUVertexAttribute attributes 

- `arrayStride`,  of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The stride, in bytes, between elements of this array.
- `stepMode`,  of type [GPUVertexStepMode](https://gpuweb.github.io#enumdef-gpuvertexstepmode), defaulting to `"vertex"`
- Whether each element of this array represents per-vertex data or per-instance data
- `attributes`,  of type sequence\<[GPUVertexAttribute](https://gpuweb.github.io#dictdef-gpuvertexattribute)>
- An array defining the layout of the vertex attributes within each element.

dictionary GPUVertexAttribute required GPUVertexFormat format required GPUSize64 offset required GPUIndex32 shaderLocation 

- `format`,  of type [GPUVertexFormat](https://gpuweb.github.io#enumdef-gpuvertexformat)
- The [GPUVertexFormat](https://gpuweb.github.io#enumdef-gpuvertexformat)
- `offset`,  of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The offset, in bytes, from the beginning of the element to the data for the attribute.
- `shaderLocation`,  of type [GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)
- The numeric location associated with this attribute, which will correspond with a
["@location" attribute](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) declared in the [vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)[module](https://gpuweb.github.io#dom-gpuprogrammablestage-module)

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`
- [GPUVertexBufferLayout](https://gpuweb.github.io#dictdef-gpuvertexbufferlayout)`descriptor`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Return `true`, if and only if, all of the following conditions are satisfied:
  - `descriptor`.[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)`device`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxVertexBufferArrayStride](https://gpuweb.github.io#dom-supported-limits-maxvertexbufferarraystride)
  - `descriptor`.[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)
  - For each attribute `attrib` in the list `descriptor`.[attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes)
    - If `descriptor`.[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)
      - `attrib`.[offset](https://gpuweb.github.io#dom-gpuvertexattribute-offset)[byteSize](https://gpuweb.github.io#abstract-opdef-gpuvertexformat-bytesize)(`attrib`.[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)`device`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxVertexBufferArrayStride](https://gpuweb.github.io#dom-supported-limits-maxvertexbufferarraystride)
Otherwise: 
      - `attrib`.[offset](https://gpuweb.github.io#dom-gpuvertexattribute-offset)[byteSize](https://gpuweb.github.io#abstract-opdef-gpuvertexformat-bytesize)(`attrib`.[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)`descriptor`.[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)
    - `attrib`.[offset](https://gpuweb.github.io#dom-gpuvertexattribute-offset)[byteSize](https://gpuweb.github.io#abstract-opdef-gpuvertexformat-bytesize)(`attrib`.[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)
    - `attrib`.[shaderLocation](https://gpuweb.github.io#dom-gpuvertexattribute-shaderlocation)`device`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxVertexAttributes](https://gpuweb.github.io#dom-supported-limits-maxvertexattributes)

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`
- [GPUVertexState](https://gpuweb.github.io#dictdef-gpuvertexstate)`descriptor`
- [GPUPipelineLayout](https://gpuweb.github.io#gpupipelinelayout)`layout`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `entryPoint` be [get the entry point](https://gpuweb.github.io#abstract-opdef-get-the-entry-point)([VERTEX](https://gpuweb.github.io#dom-gpushaderstage-vertex)`descriptor`).
2. [Assert](https://infra.spec.whatwg.org/#assert) `entryPoint` is not `null`.
3. All of the requirements in the following steps `must` be met.
  1. [validating GPUProgrammableStage](https://gpuweb.github.io#abstract-opdef-validating-gpuprogrammablestage)([VERTEX](https://gpuweb.github.io#dom-gpushaderstage-vertex)`descriptor`, `layout`, `device`) `must` succeed.
  2. `descriptor`.[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)[size](https://infra.spec.whatwg.org/#list-size) `must` be ≤ `device`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxVertexBuffers](https://gpuweb.github.io#dom-supported-limits-maxvertexbuffers)
  3. Each `vertexBuffer` layout descriptor in the list `descriptor`.[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)`must` pass [validating GPUVertexBufferLayout](https://gpuweb.github.io#abstract-opdef-validating-gpuvertexbufferlayout)(`device`, `vertexBuffer`).
  4. Let `totalEffectiveVertexAttributes` be the sum of `vertexBuffer`.[attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes)[size](https://infra.spec.whatwg.org/#list-size),
over every `vertexBuffer` in `descriptor`.[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)
  5. 
         If`device`.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
    1. If the [vertex\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-vertex_index) builtin is a [shader stage input](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-input) of `descriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)
      - Add 1 to `totalEffectiveVertexAttributes`
    2. If the [instance\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-instance_index) builtin is a [shader stage input](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-input) of `descriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)
      - Add 1 to `totalEffectiveVertexAttributes`
  6. `totalEffectiveVertexAttributes must` be ≤ `device`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxVertexAttributes](https://gpuweb.github.io#dom-supported-limits-maxvertexattributes)
  7. For every vertex attribute declaration (at location `location` with type `T`) that is [statically used](https://gpuweb.github.io#statically-used) by `entryPoint`, there `must` be exactly one pair (`i`, `j`) for which `descriptor`.[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)`i`\]?.[attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes)`j`\].[shaderLocation](https://gpuweb.github.io#dom-gpuvertexattribute-shaderlocation)`location`.
Let `attrib` be that [GPUVertexAttribute](https://gpuweb.github.io#dictdef-gpuvertexattribute)
  8. `T must` be compatible with `attrib`.[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)[vertex data type](https://gpuweb.github.io#vertex-data-type):
    - "unorm", "snorm", or "float"
    - `T` must be `f32` or `vecN<f32>`.
    - "uint"
    - `T` must be `u32` or `vecN<u32>`.
    - "sint"
    - `T` must be `i32` or `vecN<i32>`.

## 11. Copies

### 11.1. Buffer Copies

Buffer copy operations operate on raw bytes.

WebGPU provides "buffered" [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)

and "immediate" [GPUQueue](https://gpuweb.github.io#gpuqueue)

- [writeBuffer()](https://gpuweb.github.io#dom-gpuqueue-writebuffer)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[GPUBuffer](https://gpuweb.github.io#gpubuffer)

### 11.2. Texel Copies

Texel copy operations operate on texture/"image" data, rather than bytes.

WebGPU provides "buffered" [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)

and "immediate" [GPUQueue](https://gpuweb.github.io#gpuqueue)

- [writeTexture()](https://gpuweb.github.io#dom-gpuqueue-writetexture)[ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[GPUTexture](https://gpuweb.github.io#gputexture)
- [copyExternalImageToTexture()](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture)

Texel copies only guarantee that valid, finite, non-subnormal numeric values in the source have the same numeric value in the destination. Specifically, the texel block may be decoded and re-encoded in a way that preserves only those values. Where multiple byte representations are possible, the choice of representation is implementation-defined.

- Any floating-point zero value may be represented as either -0.0 or +0.0.
- Any floating-point subnormal value may be either preserved or replaced by -0.0 or +0.0.
- Any floating-point `NaN` or `Infinity` value may be replaced by an [indeterminate value](https://gpuweb.github.io/gpuweb/wgsl/#indeterminate-values).
- Packed formats and `snorm` formats may change bit-representation as long
as the represented values follow the rules above, for example:
  - `snorm` formats may represent -1.0 as either -127 or -128.
  - Formats like ["rgb9e5ufloat"](https://gpuweb.github.io#dom-gputextureformat-rgb9e5ufloat)

Note:
    For formats supporting [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)[WGSL floating point behaviors](https://gpuweb.github.io/gpuweb/wgsl/#differences-from-ieee754) may be observed.

The following definitions are used by these methods:

#### 11.2.1. `GPUTexelCopyBufferLayout`

"[GPUTexelCopyBufferLayout](https://gpuweb.github.io#gputexelcopybufferlayout)**layout**" of texels in a "**buffer**" of bytes
([GPUBuffer](https://gpuweb.github.io#gpubuffer)[AllowSharedBufferSource](https://webidl.spec.whatwg.org/#AllowSharedBufferSource)**[texel copy](https://gpuweb.github.io#texel-copy)**" operation.

dictionary GPUTexelCopyBufferLayout GPUSize64 offset GPUSize32 bytesPerRow GPUSize32 rowsPerImage 

A texel image is comprised of one or more rows of [texel blocks](https://gpuweb.github.io#texel-block), referred to here
as texel block rows. Each [texel block row](https://gpuweb.github.io#texel-block-row) of a [texel image](https://gpuweb.github.io#texel-image) must contain the
same number of [texel blocks](https://gpuweb.github.io#texel-block), and all [texel blocks](https://gpuweb.github.io#texel-block) in a [texel image](https://gpuweb.github.io#texel-image) are of the same
[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

A [GPUTexelCopyBufferLayout](https://gpuweb.github.io#gputexelcopybufferlayout)[texel images](https://gpuweb.github.io#texel-image) within some linear memory.
It’s used when copying data between a [texture](https://gpuweb.github.io#texture) and a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[texture](https://gpuweb.github.io#texture) from the [GPUQueue](https://gpuweb.github.io#gpuqueue)

- For [2d](https://gpuweb.github.io#dom-gputexturedimension-2d)[texel images](https://gpuweb.github.io#texel-image) and [array layers](https://gpuweb.github.io#array-layer).
- For [3d](https://gpuweb.github.io#dom-gputexturedimension-3d)[texel images](https://gpuweb.github.io#texel-image) and depth [slices](https://gpuweb.github.io#slice).

Operations that copy between byte arrays and textures always operate on whole [texel block](https://gpuweb.github.io#texel-block).
It’s not possible to update only a part of a [texel block](https://gpuweb.github.io#texel-block).

[Texel blocks](https://gpuweb.github.io#texel-block) are tightly packed within each [texel block row](https://gpuweb.github.io#texel-block-row) in the linear memory layout of a
[texel copy](https://gpuweb.github.io#texel-copy), with each subsequent [texel block](https://gpuweb.github.io#texel-block) immediately following the previous [texel block](https://gpuweb.github.io#texel-block),
with no padding.
This includes [copies](https://gpuweb.github.io#copying-depth-stencil) to/from specific aspects of [depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format) textures:
stencil values are tightly packed in an array of bytes;
depth values are tightly packed in an array of the appropriate type ("depth16unorm" or "depth32float").

- `offset`,  of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64), defaulting to `0`
- The offset, in bytes, from the beginning of the texel data source (such as a
[GPUTexelCopyBufferInfo.buffer](https://gpuweb.github.io#dom-gputexelcopybufferinfo-buffer)
- `bytesPerRow`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- The stride, in bytes, between the beginning of each [texel block row](https://gpuweb.github.io#texel-block-row) and the subsequent [texel block row](https://gpuweb.github.io#texel-block-row).
Required if there are multiple [texel block rows](https://gpuweb.github.io#texel-block-row) (i.e. the copy height or depth is more
than one block).
- `rowsPerImage`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- Number of [texel block rows](https://gpuweb.github.io#texel-block-row) per single [texel image](https://gpuweb.github.io#texel-image) of the [texture](https://gpuweb.github.io#texture). [rowsPerImage](https://gpuweb.github.io#dom-gputexelcopybufferlayout-rowsperimage)[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)[texel image](https://gpuweb.github.io#texel-image) of data and the subsequent [texel image](https://gpuweb.github.io#texel-image).
Required if there are multiple [texel images](https://gpuweb.github.io#texel-image) (i.e. the copy depth is more than one).

#### 11.2.2. `GPUTexelCopyBufferInfo`

"[GPUTexelCopyBufferInfo](https://gpuweb.github.io#gputexelcopybufferinfo)**info**" ([GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUTexelCopyBufferLayout](https://gpuweb.github.io#gputexelcopybufferlayout)**buffer**" source or destination of a "**[texel copy](https://gpuweb.github.io#texel-copy)**" operation.
Together with the `copySize`, it describes the footprint of a region of texels in a [GPUBuffer](https://gpuweb.github.io#gpubuffer)

dictionary GPUTexelCopyBufferInfo GPUTexelCopyBufferLayout required GPUBuffer buffer 

- `buffer`,  of type [GPUBuffer](https://gpuweb.github.io#gpubuffer)
- A buffer which either contains texel data to be copied or will store the texel data being copied, depending on the method it is being passed to.

**Arguments:**

- [GPUTexelCopyBufferInfo](https://gpuweb.github.io#gputexelcopybufferinfo)`imageCopyBuffer`

**Returns:** [boolean](https://webidl.spec.whatwg.org/#idl-boolean)

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Return `true` if and only if all of the following conditions are satisfied:
  - `imageCopyBuffer`.[buffer](https://gpuweb.github.io#dom-gputexelcopybufferinfo-buffer)[valid](https://w3c.github.io/i18n-glossary/#dfn-valid) [GPUBuffer](https://gpuweb.github.io#gpubuffer)
  - `imageCopyBuffer`.[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)

#### 11.2.3. `GPUTexelCopyTextureInfo`

"[GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)**info**" ([GPUTexture](https://gpuweb.github.io#gputexture)**texture**" source or destination of a "**[texel copy](https://gpuweb.github.io#texel-copy)**" operation.
Together with the `copySize`, it describes a sub-region of a texture
(spanning one or more contiguous [texture subresources](https://gpuweb.github.io#texture-subresources) at the same mip-map level).

dictionary GPUTexelCopyTextureInfo required GPUTexture texture GPUIntegerCoordinate mipLevel GPUOrigin3D origin GPUTextureAspect aspect 

- `texture`,  of type [GPUTexture](https://gpuweb.github.io#gputexture)
- Texture to copy to/from.
- `mipLevel`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate), defaulting to `0`
- Mip-map level of the [texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
- `origin`,  of type [GPUOrigin3D](https://gpuweb.github.io#typedefdef-gpuorigin3d), defaulting to `{}`
- Defines the origin of the copy - the minimum corner of the texture sub-region to copy to/from.
Together with `copySize`, defines the full copy sub-region.
- `aspect`,  of type [GPUTextureAspect](https://gpuweb.github.io#enumdef-gputextureaspect), defaulting to `"all"`
- Defines which aspects of the [texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)

`index`of

[GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)
`copyTexture`is determined by running the following steps:

[GPUTexelCopyBufferLayout](https://gpuweb.github.io#gputexelcopybufferlayout)
`bufferLayout`corresponding to

[texel block](https://gpuweb.github.io#texel-block)

`x`,

`y`of depth slice or array layer

`z`of a

[GPUTexture](https://gpuweb.github.io#gputexture)
`texture`is determined by running the following steps:

1. Let `blockBytes` be the [texel block copy footprint](https://gpuweb.github.io#texel-block-copy-footprint) of `texture`.[format](https://gpuweb.github.io#dom-gputexture-format)
2. Let `imageOffset` be (`z` × `bufferLayout`.[rowsPerImage](https://gpuweb.github.io#dom-gputexelcopybufferlayout-rowsperimage)`bufferLayout`.[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)`bufferLayout`.[offset](https://gpuweb.github.io#dom-gputexelcopybufferlayout-offset)
3. Let `rowOffset` be (`y` × `bufferLayout`.[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)`imageOffset`.
4. Let `blockOffset` be (`x` × `blockBytes`) + `rowOffset`.
5. Return `blockOffset`.

`texelCopyTextureInfo`,

`copySize`)

**Arguments:**

- [GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)`texelCopyTextureInfo`
- [GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)`copySize`

**Returns:** [boolean](https://webidl.spec.whatwg.org/#idl-boolean)

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `blockWidth` be the [texel block width](https://gpuweb.github.io#texel-block-width) of `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[format](https://gpuweb.github.io#dom-gputexture-format)
2. Let `blockHeight` be the [texel block height](https://gpuweb.github.io#texel-block-height) of `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[format](https://gpuweb.github.io#dom-gputexture-format)
3. Return `true` if and only if all of the following conditions apply:
  - [validating texture copy range](https://gpuweb.github.io#abstract-opdef-validating-texture-copy-range)(`texelCopyTextureInfo`, `copySize`) returns `true`.
  - `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[valid](https://w3c.github.io/i18n-glossary/#dfn-valid) [GPUTexture](https://gpuweb.github.io#gputexture)
  - `texelCopyTextureInfo`.[mipLevel](https://gpuweb.github.io#dom-gputexelcopytextureinfo-miplevel)`texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[mipLevelCount](https://gpuweb.github.io#dom-gputexture-miplevelcount)
  - `texelCopyTextureInfo`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)[x](https://gpuweb.github.io#gpuorigin3d-x) must be a multiple of `blockWidth`.
  - `texelCopyTextureInfo`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)[y](https://gpuweb.github.io#gpuorigin3d-y) must be a multiple of `blockHeight`.
  - The [GPUTexelCopyTextureInfo physical subresource size](https://gpuweb.github.io#gputexelcopytextureinfo-physical-subresource-size) of `texelCopyTextureInfo` is equal to `copySize` if either of
the following conditions is true:
    - `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[format](https://gpuweb.github.io#dom-gputexture-format)
    - `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)

`texelCopyTextureInfo`,

`bufferLayout`,

`dataLength`,

`copySize`,

`textureUsage`,

`aligned`)

**Arguments:**

- [GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)`texelCopyTextureInfo`
- [GPUTexelCopyBufferLayout](https://gpuweb.github.io#gputexelcopybufferlayout)`bufferLayout`
- [GPUSize64Out](https://gpuweb.github.io#typedefdef-gpusize64out)`dataLength`
- [GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)`copySize`
- [GPUTextureUsage](https://gpuweb.github.io#namespacedef-gputextureusage)`textureUsage`
- [boolean](https://webidl.spec.whatwg.org/#idl-boolean)`aligned`

**Returns:** [boolean](https://webidl.spec.whatwg.org/#idl-boolean)

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `texture` be `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
2. Let `aspectSpecificFormat` = `texture`.[format](https://gpuweb.github.io#dom-gputexture-format)
3. Let `offsetAlignment` = [texel block copy footprint](https://gpuweb.github.io#texel-block-copy-footprint) of `texture`.[format](https://gpuweb.github.io#dom-gputexture-format)
4. Return `true` if and only if all of the following conditions apply:
  1. [validating GPUTexelCopyTextureInfo](https://gpuweb.github.io#abstract-opdef-validating-gputexelcopytextureinfo)(`texelCopyTextureInfo`, `copySize`) returns `true`.
  2. `texture`.[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)
  3. `texture`.[usage](https://gpuweb.github.io#dom-gputexture-usage)`textureUsage`.
  4. If `texture`.[format](https://gpuweb.github.io#dom-gputexture-format)[depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format) format:
    1. `texelCopyTextureInfo`.[aspect](https://gpuweb.github.io#dom-gputexelcopytextureinfo-aspect)`texture`.[format](https://gpuweb.github.io#dom-gputexture-format)
    2. If `textureUsage` is:
      - [COPY_SRC](https://gpuweb.github.io#dom-gputextureusage-copy_src)
      - That aspect must be a valid [texel copy](https://gpuweb.github.io#texel-copy) source according to [§ 26.1.2 Depth-stencil formats](https://gpuweb.github.io#depth-formats).
      - [COPY_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)
      - That aspect must be a valid [texel copy](https://gpuweb.github.io#texel-copy) destination according to [§ 26.1.2 Depth-stencil formats](https://gpuweb.github.io#depth-formats).
    3. Set `aspectSpecificFormat` to the [aspect-specific format](https://gpuweb.github.io#aspect-specific-format) according to [§ 26.1.2 Depth-stencil formats](https://gpuweb.github.io#depth-formats).
    4. Set `offsetAlignment` to 4.
  5. If `aligned` is `true`:
    1. `bufferLayout`.[offset](https://gpuweb.github.io#dom-gputexelcopybufferlayout-offset)`offsetAlignment`.
  6. [validating linear texture data](https://gpuweb.github.io#abstract-opdef-validating-linear-texture-data)(`bufferLayout`, `dataLength`, `aspectSpecificFormat`, `copySize`) succeeds.

#### 11.2.4. `GPUCopyExternalImageDestInfo`

WebGPU textures hold raw numeric data, and are not tagged with semantic metadata describing colors.
However, [copyExternalImageToTexture()](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture)

"[GPUCopyExternalImageDestInfo](https://gpuweb.github.io#gpucopyexternalimagedestinfo)**info**" about the "**dest**ination" of a
"" operation.
It is a [GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)

dictionary GPUCopyExternalImageDestInfo GPUTexelCopyTextureInfo PredefinedColorSpace colorSpace boolean premultipliedAlpha false ; };

- `colorSpace`,  of type [PredefinedColorSpace](https://html.spec.whatwg.org/multipage/canvas.html#predefinedcolorspace), defaulting to `"srgb"`
- Describes the color space and encoding used to encode data into the destination texture. This [may result](https://gpuweb.github.io#color-space-conversions) in values outside of the range \[0, 1\]
being written to the target texture, if its format can represent them.
Otherwise, the results are clamped to the target texture format’s range.
Note:
If [colorSpace](https://gpuweb.github.io#dom-gpucopyexternalimagedestinfo-colorspace)[§ 3.11.2 Color Space Conversion Elision](https://gpuweb.github.io#color-space-conversion-elision).
- `premultipliedAlpha`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- Describes whether the data written into the texture should have its RGB channels premultiplied by the alpha channel, or not. If this option is set to `true` and the [source](https://gpuweb.github.io#dom-gpucopyexternalimagesourceinfo-source)
Note:
If [premultipliedAlpha](https://gpuweb.github.io#dom-gpucopyexternalimagedestinfo-premultipliedalpha)[§ 3.11.2 Color Space Conversion Elision](https://gpuweb.github.io#color-space-conversion-elision).

#### 11.2.5. `GPUCopyExternalImageSourceInfo`

"[GPUCopyExternalImageSourceInfo](https://gpuweb.github.io#gpucopyexternalimagesourceinfo)**info**" about the "**source**" of a
"" operation.

typedef (ImageBitmap or ImageData or HTMLImageElement or HTMLVideoElement or VideoFrame or HTMLCanvasElement or OffscreenCanvas GPUCopyExternalImageSource dictionary GPUCopyExternalImageSourceInfo required GPUCopyExternalImageSource source GPUOrigin2D origin boolean flipY false ; };

[GPUCopyExternalImageSourceInfo](https://gpuweb.github.io#gpucopyexternalimagesourceinfo)

- `source`,  of type [GPUCopyExternalImageSource](https://gpuweb.github.io#typedefdef-gpucopyexternalimagesource)
- The source of the [texel copy](https://gpuweb.github.io#texel-copy). The copy source data is captured at the moment that [copyExternalImageToTexture()](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture)[external source dimensions](https://gpuweb.github.io#external-source-dimensions) table.
- `origin`,  of type [GPUOrigin2D](https://gpuweb.github.io#typedefdef-gpuorigin2d), defaulting to `{}`
- Defines the origin of the copy - the minimum (top-left) corner of the source sub-region to copy from.
Together with `copySize`, defines the full copy sub-region.
- `flipY`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- Describes whether the source image is vertically flipped, or not. If this option is set to `true`, the copy is flipped vertically: the bottom row of the source
region is copied into the first row of the destination region, and so on.
The [origin](https://gpuweb.github.io#dom-gpucopyexternalimagesourceinfo-origin)

When external sources are used when creating or copying to textures, the external source dimensions are defined by the source type, given by this table:

#### 11.2.6. Subroutines

**Arguments:**

- [GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)`texelCopyTextureInfo`

**Returns:** [GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)

The [GPUTexelCopyTextureInfo physical subresource size](https://gpuweb.github.io#gputexelcopytextureinfo-physical-subresource-size) of `texelCopyTextureInfo` is calculated as follows:

Its [width](https://gpuweb.github.io#gpuextent3d-width), [height](https://gpuweb.github.io#gpuextent3d-height) and [depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) are the width, height, and depth, respectively,
    of the [physical miplevel-specific texture extent](https://gpuweb.github.io#physical-miplevel-specific-texture-extent) of `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[subresource](https://gpuweb.github.io#subresource) at [mipmap level](https://gpuweb.github.io#mipmap-level)
    `texelCopyTextureInfo`.[mipLevel](https://gpuweb.github.io#dom-gputexelcopytextureinfo-miplevel)

**Arguments:**

- [GPUTexelCopyBufferLayout](https://gpuweb.github.io#gputexelcopybufferlayout)
`layout`
- Layout of the linear texture data.
- [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
`byteSize`
- Total size of the linear data, in bytes.
- [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
`format`
- Format of the texture.
- [GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
`copyExtent`
- Extent of the texture to copy.

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let: 
  - `widthInBlocks` be `copyExtent`.[width](https://gpuweb.github.io#gpuextent3d-width) ÷ the [texel block width](https://gpuweb.github.io#texel-block-width) of `format`. [Assert](https://infra.spec.whatwg.org/#assert) this is an integer.
  - `heightInBlocks` be `copyExtent`.[height](https://gpuweb.github.io#gpuextent3d-height) ÷ the [texel block height](https://gpuweb.github.io#texel-block-height) of `format`. [Assert](https://infra.spec.whatwg.org/#assert) this is an integer.
  - `bytesInLastRow` be `widthInBlocks` × the [texel block copy footprint](https://gpuweb.github.io#texel-block-copy-footprint) of `format`.
2. Fail if the following input validation requirements are not met: 
  - If `heightInBlocks` > 1, `layout`.[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)
  - If `copyExtent`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) > 1, `layout`.[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)`layout`.[rowsPerImage](https://gpuweb.github.io#dom-gputexelcopybufferlayout-rowsperimage)
  - If specified, `layout`.[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)`bytesInLastRow`.
  - If specified, `layout`.[rowsPerImage](https://gpuweb.github.io#dom-gputexelcopybufferlayout-rowsperimage)`heightInBlocks`.
3. Let: 
  - `bytesPerRow` be `layout`.[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)
  - `rowsPerImage` be `layout`.[rowsPerImage](https://gpuweb.github.io#dom-gputexelcopybufferlayout-rowsperimage)
Note: These default values have no effect, as they’re always multiplied by 0.
4. Let `requiredBytesInCopy` be 0.
5. If `copyExtent`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) > 0:
  1. Increment `requiredBytesInCopy` by `bytesPerRow` × `rowsPerImage` × (`copyExtent`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) − 1).
  2. If `heightInBlocks` > 0:
    1. Increment `requiredBytesInCopy` by `bytesPerRow` × (`heightInBlocks` − 1) + `bytesInLastRow`.
6. Fail if the following condition is not satisfied: 
  - The layout fits inside the linear data:
`layout`.[offset](https://gpuweb.github.io#dom-gputexelcopybufferlayout-offset)`requiredBytesInCopy` ≤ `byteSize`.

**Arguments:**

- [GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)
`texelCopyTextureInfo`
- The texture subresource being copied into and copy origin.
- [GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
`copySize`
- The size of the texture.

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `blockWidth` be the [texel block width](https://gpuweb.github.io#texel-block-width) of `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[format](https://gpuweb.github.io#dom-gputexture-format)
2. Let `blockHeight` be the [texel block height](https://gpuweb.github.io#texel-block-height) of `texelCopyTextureInfo`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[format](https://gpuweb.github.io#dom-gputexture-format)
3. Let `subresourceSize` be the [GPUTexelCopyTextureInfo physical subresource size](https://gpuweb.github.io#gputexelcopytextureinfo-physical-subresource-size) of `texelCopyTextureInfo`.
4. Return whether all the conditions below are satisfied: 
  - (`texelCopyTextureInfo`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)[x](https://gpuweb.github.io#gpuorigin3d-x) + `copySize`.[width](https://gpuweb.github.io#gpuextent3d-width)) ≤ `subresourceSize`.[width](https://gpuweb.github.io#gpuextent3d-width)
  - (`texelCopyTextureInfo`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)[y](https://gpuweb.github.io#gpuorigin3d-y) + `copySize`.[height](https://gpuweb.github.io#gpuextent3d-height)) ≤ `subresourceSize`.[height](https://gpuweb.github.io#gpuextent3d-height)
  - (`texelCopyTextureInfo`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)[z](https://gpuweb.github.io#gpuorigin3d-z) + `copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers)) ≤ `subresourceSize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers)
  - `copySize`.[width](https://gpuweb.github.io#gpuextent3d-width) must be a multiple of `blockWidth`.
  - `copySize`.[height](https://gpuweb.github.io#gpuextent3d-height) must be a multiple of `blockHeight`.
Note:
    The texture copy range is validated against the *physical* (rounded-up)
    size for [compressed formats](https://gpuweb.github.io#compressed-format), allowing copies to access texture
    blocks which are not fully inside the texture.

[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
`format1`and

`format2`are copy-compatible if:

- `format1` equals `format2`, or
- `format1` and `format2` differ only in whether they are `srgb` formats (have the `-srgb` suffix).

`texelCopyTextureInfo`,

`copySize`) is the subset of subresources of

`texture`=

`texelCopyTextureInfo`.

[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
`s`satisfies the following:

- The [mipmap level](https://gpuweb.github.io#mipmap-level) of `s` equals `texelCopyTextureInfo`.[mipLevel](https://gpuweb.github.io#dom-gputexelcopytextureinfo-miplevel)
- The [aspect](https://gpuweb.github.io#aspect) of `s` is in the [set of aspects](https://gpuweb.github.io#gputextureaspect-set-of-aspects) of `texelCopyTextureInfo`.[aspect](https://gpuweb.github.io#dom-gputexelcopytextureinfo-aspect)
- If `texture`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
  - The [array layer](https://gpuweb.github.io#array-layer) of `s` is ≥ `texelCopyTextureInfo`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)[z](https://gpuweb.github.io#gpuorigin3d-z) and \< `texelCopyTextureInfo`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)[z](https://gpuweb.github.io#gpuorigin3d-z) + `copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers).

## 12. Command Buffers

Command buffers are pre-recorded lists of [GPU commands](https://gpuweb.github.io#gpu-command) (blocks of [queue timeline](https://gpuweb.github.io#queue-timeline)
steps) that can be submitted to a [GPUQueue](https://gpuweb.github.io#gpuqueue)[queue timeline](https://gpuweb.github.io#queue-timeline), such as setting state, drawing, copying resources, etc.

A [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)[invalidated](https://gpuweb.github.io#abstract-opdef-invalidate).
To reuse rendering commands across multiple submissions, use [GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)

### 12.1. `GPUCommandBuffer`

\[Exposed Window ,Worker ),SecureContext interface GPUCommandBuffer GPUCommandBuffer includes GPUObjectBase 

[GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[command_list]]`, of type [list](https://infra.spec.whatwg.org/#list)\<[GPU command](https://gpuweb.github.io#gpu-command)>, readonly
- A [list](https://infra.spec.whatwg.org/#list) of [GPU commands](https://gpuweb.github.io#gpu-command) to be executed on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when this command
buffer is submitted.
- `[[renderState]]`, of type [RenderState](https://gpuweb.github.io#renderstate), initially `null`
- The current state used by any render pass commands being executed.
- `[[used_bind_groups]]`, of type [set](https://infra.spec.whatwg.org/#ordered-set)\<[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
- A [set](https://infra.spec.whatwg.org/#ordered-set) of all [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)

#### 12.1.1. Command Buffer Creation

dictionary GPUCommandBufferDescriptor GPUObjectDescriptorBase 

## 13. Command Encoding

### 13.1. `GPUCommandsMixin`

[GPUCommandsMixin](https://gpuweb.github.io#gpucommandsmixin)

interface mixin GPUCommandsMixin 

[GPUCommandsMixin](https://gpuweb.github.io#gpucommandsmixin)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[state]]`, of type [encoder state](https://gpuweb.github.io#encoder-state), initially "[open](https://gpuweb.github.io#encoder-state-open)"
- The current state of the encoder.
- `[[commands]]`, of type [list](https://infra.spec.whatwg.org/#list)\<[GPU command](https://gpuweb.github.io#gpu-command)>, initially `[]`
- A [list](https://infra.spec.whatwg.org/#list) of [GPU commands](https://gpuweb.github.io#gpu-command) to be executed on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when a [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)
- `[[used_bind_groups]]`, of type [set](https://infra.spec.whatwg.org/#ordered-set)\<[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
- A [set](https://infra.spec.whatwg.org/#ordered-set) of all [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[setBindGroup()](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup) during command encoding.

The encoder state may be one of the following:

- "open"
- The encoder is available to encode new commands.
- "locked"
- The encoder cannot be used, because it is locked by a child encoder: it is a
[GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)[open](https://gpuweb.github.io#encoder-state-open)" again when the pass is ended.
Any command issued in this state [invalidates](https://gpuweb.github.io#abstract-opdef-invalidate) the encoder.
- "ended"
- The encoder has been ended and new commands can no longer be encoded. Any command issued in this state will [generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).

[GPUCommandsMixin](https://gpuweb.github.io#gpucommandsmixin)
`encoder`run the

following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. If `encoder`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)
  - "[open](https://gpuweb.github.io#encoder-state-open)"
  - Return `true`.
  - "[locked](https://gpuweb.github.io#encoder-state-locked)"
  - [Invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `encoder` and return `false`.
  - "[ended](https://gpuweb.github.io#encoder-state-ended)"
  - [Generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), and return `false`.

[GPUCommandsMixin](https://gpuweb.github.io#gpucommandsmixin)
`encoder`which issues the steps of a

[GPU Command](https://gpuweb.github.io#gpu-command)

`command`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. [Append](https://infra.spec.whatwg.org/#list-append) `command` to `encoder`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)
2. When `command` is executed as part of a [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)
  1. Issue the steps of `command`.

### 13.2. `GPUCommandEncoder`

\[Exposed Window ,Worker ),SecureContext interface GPUCommandEncoder GPURenderPassEncoder beginRenderPass GPURenderPassDescriptor descriptor GPUComputePassEncoder beginComputePass optional GPUComputePassDescriptor descriptor undefined copyBufferToBuffer GPUBuffer source GPUBuffer destination optional GPUSize64 size undefined copyBufferToBuffer GPUBuffer source GPUSize64 sourceOffset GPUBuffer destination GPUSize64 destinationOffset optional GPUSize64 size undefined copyBufferToTexture GPUTexelCopyBufferInfo source GPUTexelCopyTextureInfo destination GPUExtent3D copySize undefined copyTextureToBuffer GPUTexelCopyTextureInfo source GPUTexelCopyBufferInfo destination GPUExtent3D copySize undefined copyTextureToTexture GPUTexelCopyTextureInfo source GPUTexelCopyTextureInfo destination GPUExtent3D copySize undefined clearBuffer GPUBuffer buffer optional GPUSize64 offset optional GPUSize64 size undefined resolveQuerySet GPUQuerySet querySet GPUSize32 firstQuery GPUSize32 queryCount GPUBuffer destination GPUSize64 destinationOffset GPUCommandBuffer finish optional GPUCommandBufferDescriptor descriptor GPUCommandEncoder includes GPUObjectBase GPUCommandEncoder includes GPUCommandsMixin GPUCommandEncoder includes GPUDebugCommandsMixin 

#### 13.2.1. Command Encoder Creation

dictionary GPUCommandEncoderDescriptor GPUObjectDescriptorBase 

- `createCommandEncoder(descriptor)`
- Creates a [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
**Arguments:**
Arguments for the [GPUDevice.createCommandEncoder(descriptor)](https://gpuweb.github.io#dom-gpudevice-createcommandencoder) method.Parameter Type Nullable Optional Description `descriptor`
[GPUCommandEncoderDescriptor](https://gpuweb.github.io#dictdef-gpucommandencoderdescriptor)
✘ ✔ Description of the [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
**Returns:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `e` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)`descriptor`).
  2. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  3. Return `e`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `e` and return.
    - `this` must not be [lost](https://gpuweb.github.io#abstract-opdef-invalid).

[Creating a](https://gpuweb.github.io#example-ee4e6506)

[GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)[GPUQueue](https://gpuweb.github.io#gpuqueue)
const commandEncoder= gpuDevice. createCommandEncoder(); commandEncoder. clearBuffer( buffer); const commandBuffer= commandEncoder. finish(); gpuDevice. queue. submit(\[ commandBuffer\]); 

### 13.3. Pass Encoding

- `beginRenderPass(descriptor)`
- Begins encoding a render pass described by `descriptor`.
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
`this`. **Arguments:**
Arguments for the [GPUCommandEncoder.beginRenderPass(descriptor)](https://gpuweb.github.io#dom-gpucommandencoder-beginrenderpass) method.Parameter Type Nullable Optional Description `descriptor`
[GPURenderPassDescriptor](https://gpuweb.github.io#dictdef-gpurenderpassdescriptor)
✘ ✘ Description of the [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
**Returns:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. For each non-`null` `colorAttachment` in `descriptor`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
    1. If `colorAttachment`.[clearValue](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-clearvalue)[provided](https://infra.spec.whatwg.org/#map-exists):
      1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUColor shape](https://gpuweb.github.io#abstract-opdef-validate-gpucolor-shape)(`colorAttachment`.[clearValue](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-clearvalue)
  2. Let `pass` be a new [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
  3. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  4. Return `pass`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`.
If it returns false, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pass` and return.
  2. Set `this`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[locked](https://gpuweb.github.io#encoder-state-locked)".
  3. Let `attachmentRegions` be a [list](https://infra.spec.whatwg.org/#list) of \[[texture subresource](https://gpuweb.github.io#texture-subresources), [depthSlice](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-depthslice)["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
  4. For each non-`null` `colorAttachment` in `descriptor`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
    1. Add \[`colorAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)`colorAttachment`.[depthSlice](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-depthslice)`null`\] to `attachmentRegions`.
    2. If `colorAttachment`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)`null`:
      1. Add \[`colorAttachment`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)`undefined`\] to `attachmentRegions`.
  5. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pass` and return.
    - `descriptor` must meet the [Valid Usage](https://gpuweb.github.io#abstract-opdef-gpurenderpassdescriptor-valid-usage) rules
given device `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
    - The set of texture regions in `attachmentRegions` must be pairwise disjoint.
That is, no two texture regions may overlap.
  6. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) each [texture subresource](https://gpuweb.github.io#texture-subresources) in `attachmentRegions` to `pass`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[attachment](https://gpuweb.github.io#internal-usage-attachment).
  7. Let `depthStencilAttachment` be `descriptor`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
  8. If `depthStencilAttachment` is not `null`:
    1. Let `depthStencilView` be `depthStencilAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
    2. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) the [depth](https://gpuweb.github.io#aspect-depth) [subresource](https://gpuweb.github.io#subresource) of `depthStencilView`, if any,
to `pass`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[attachment-read](https://gpuweb.github.io#internal-usage-attachment-read) if `depthStencilAttachment`.[depthReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthreadonly)[attachment](https://gpuweb.github.io#internal-usage-attachment) otherwise.
    3. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) the [stencil](https://gpuweb.github.io#aspect-stencil) [subresource](https://gpuweb.github.io#subresource) of `depthStencilView`, if any,
to `pass`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[attachment-read](https://gpuweb.github.io#internal-usage-attachment-read) if `depthStencilAttachment`.[stencilReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilreadonly)[attachment](https://gpuweb.github.io#internal-usage-attachment) otherwise.
    4. Set `pass`.[\[\[depthReadOnly\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-depthreadonly-slot)`depthStencilAttachment`.[depthReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthreadonly)
    5. Set `pass`.[\[\[stencilReadOnly\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-stencilreadonly-slot)`depthStencilAttachment`.[stencilReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilreadonly)
  9. Set `pass`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-layout-slot)[derive render targets layout from pass](https://gpuweb.github.io#abstract-opdef-derive-render-targets-layout-from-pass)(`descriptor`).
  10. If `descriptor`.[timestampWrites](https://gpuweb.github.io#dom-gpurenderpassdescriptor-timestampwrites)[provided](https://infra.spec.whatwg.org/#map-exists):
    1. Let `timestampWrites` be `descriptor`.[timestampWrites](https://gpuweb.github.io#dom-gpurenderpassdescriptor-timestampwrites)
    2. If `timestampWrites`.[beginningOfPassWriteIndex](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-beginningofpasswriteindex)[provided](https://infra.spec.whatwg.org/#map-exists), [append](https://infra.spec.whatwg.org/#list-append) a [GPU command](https://gpuweb.github.io#gpu-command) to `this`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)
      1. Before the pass commands begin executing,
write the [current queue timestamp](https://gpuweb.github.io#abstract-opdef-current-queue-timestamp) into index `timestampWrites`.[beginningOfPassWriteIndex](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-beginningofpasswriteindex)`timestampWrites`.[querySet](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-queryset)
    3. If `timestampWrites`.[endOfPassWriteIndex](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-endofpasswriteindex)[provided](https://infra.spec.whatwg.org/#map-exists), set `pass`.[\[\[endTimestampWrite\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-endtimestampwrite-slot)[GPU command](https://gpuweb.github.io#gpu-command) with the following steps:
      1. After the pass commands finish executing,
write the [current queue timestamp](https://gpuweb.github.io#abstract-opdef-current-queue-timestamp) into index `timestampWrites`.[endOfPassWriteIndex](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-endofpasswriteindex)`timestampWrites`.[querySet](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-queryset)
  11. Set `pass`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)
  12. Set `pass`.[\[\[maxDrawCount\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-maxdrawcount-slot)`descriptor`.[maxDrawCount](https://gpuweb.github.io#dom-gpurenderpassdescriptor-maxdrawcount)
  13. Set `pass`.[\[\[maxDrawCount\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-maxdrawcount-slot)`descriptor`.[maxDrawCount](https://gpuweb.github.io#dom-gpurenderpassdescriptor-maxdrawcount)
  14. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let the [\[\[renderState\]\]](https://gpuweb.github.io#dom-gpucommandbuffer-renderstate-slot)[GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)[RenderState](https://gpuweb.github.io#renderstate).
  2. Set [\[\[renderState\]\]](https://gpuweb.github.io#dom-gpucommandbuffer-renderstate-slot)[\[\[colorAttachments\]\]](https://gpuweb.github.io#dom-renderstate-colorattachments-slot)`descriptor`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
  3. Set [\[\[renderState\]\]](https://gpuweb.github.io#dom-gpucommandbuffer-renderstate-slot)[\[\[depthStencilAttachment\]\]](https://gpuweb.github.io#dom-renderstate-depthstencilattachment-slot)`descriptor`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
  4. For each non-`null` `colorAttachment` in `descriptor`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
    1. Let `colorView` be `colorAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)
    2. If `colorView`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
      - ["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
      - Let `colorSubregion` be `colorAttachment`.[depthSlice](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-depthslice)`colorView`.
      - Otherwise
      - Let `colorSubregion` be `colorView`.
    3. If `colorAttachment`.[loadOp](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-loadop)
      - ["load"](https://gpuweb.github.io#dom-gpuloadop-load)
      - Ensure the contents of `colorSubregion` are loaded into the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with `colorSubregion`.
      - ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
      - Set every [texel](https://gpuweb.github.io#texel-block) of the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with `colorSubregion` to `colorAttachment`.[clearValue](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-clearvalue)
  5. If `depthStencilAttachment` is not `null`:
    1. If `depthStencilAttachment`.[depthLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthloadop)
      - Not [provided](https://infra.spec.whatwg.org/#map-exists)
      - [Assert](https://infra.spec.whatwg.org/#assert) that `depthStencilAttachment`.[depthReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthreadonly)`true` and ensure the contents of the [depth](https://gpuweb.github.io#aspect-depth) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` are loaded into the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with `depthStencilView`.
      - ["load"](https://gpuweb.github.io#dom-gpuloadop-load)
      - Ensure the contents of the [depth](https://gpuweb.github.io#aspect-depth) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` are loaded into the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with `depthStencilView`.
      - ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
      - Set every [texel](https://gpuweb.github.io#texel-block) of the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with the [depth](https://gpuweb.github.io#aspect-depth) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` to `depthStencilAttachment`.[depthClearValue](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthclearvalue)
    2. If `depthStencilAttachment`.[stencilLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilloadop)
      - Not [provided](https://infra.spec.whatwg.org/#map-exists)
      - [Assert](https://infra.spec.whatwg.org/#assert) that `depthStencilAttachment`.[stencilReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilreadonly)`true` and ensure the contents of the [stencil](https://gpuweb.github.io#aspect-stencil) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` are loaded into the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with `depthStencilView`.
      - ["load"](https://gpuweb.github.io#dom-gpuloadop-load)
      - Ensure the contents of the [stencil](https://gpuweb.github.io#aspect-stencil) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` are loaded into the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with `depthStencilView`.
      - ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
      - Set every [texel](https://gpuweb.github.io#texel-block) of the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with the [stencil](https://gpuweb.github.io#aspect-stencil) [subresource](https://gpuweb.github.io#gputextureview-subresources) `depthStencilView` to `depthStencilAttachment`.[stencilClearValue](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilclearvalue)
Note: [Read-only depth-stencil](https://gpuweb.github.io#read-only-depth-stencil) attachments are implicitly treated as though the ["load"](https://gpuweb.github.io#dom-gpuloadop-load)[GPURenderPassDepthStencilAttachment Valid Usage](https://gpuweb.github.io#abstract-opdef-gpurenderpassdepthstencilattachment-gpurenderpassdepthstencilattachment-valid-usage).
- `beginComputePass(descriptor)`
- Begins encoding a compute pass described by `descriptor`.
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
`this`. **Arguments:**
Arguments for the [GPUCommandEncoder.beginComputePass(descriptor)](https://gpuweb.github.io#dom-gpucommandencoder-begincomputepass) method.Parameter Type Nullable Optional Description `descriptor` [GPUComputePassDescriptor](https://gpuweb.github.io#dictdef-gpucomputepassdescriptor)
✘ ✔ **Returns:** [GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `pass` be a new [GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)
  2. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  3. Return `pass`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`.
If it returns false, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pass` and return.
  2. Set `this`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[locked](https://gpuweb.github.io#encoder-state-locked)".
  3. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `pass` and return.
    - If `descriptor`.[timestampWrites](https://gpuweb.github.io#dom-gpucomputepassdescriptor-timestampwrites)[provided](https://infra.spec.whatwg.org/#map-exists):
      - [Validate timestampWrites](https://gpuweb.github.io#abstract-opdef-validate-timestampwrites)(`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)`descriptor`.[timestampWrites](https://gpuweb.github.io#dom-gpucomputepassdescriptor-timestampwrites)
  4. If `descriptor`.[timestampWrites](https://gpuweb.github.io#dom-gpucomputepassdescriptor-timestampwrites)[provided](https://infra.spec.whatwg.org/#map-exists):
    1. Let `timestampWrites` be `descriptor`.[timestampWrites](https://gpuweb.github.io#dom-gpucomputepassdescriptor-timestampwrites)
    2. If `timestampWrites`.[beginningOfPassWriteIndex](https://gpuweb.github.io#dom-gpucomputepasstimestampwrites-beginningofpasswriteindex)[provided](https://infra.spec.whatwg.org/#map-exists), [append](https://infra.spec.whatwg.org/#list-append) a [GPU command](https://gpuweb.github.io#gpu-command) to `this`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)
      1. Before the pass commands begin executing,
write the [current queue timestamp](https://gpuweb.github.io#abstract-opdef-current-queue-timestamp) into index `timestampWrites`.[beginningOfPassWriteIndex](https://gpuweb.github.io#dom-gpucomputepasstimestampwrites-beginningofpasswriteindex)`timestampWrites`.[querySet](https://gpuweb.github.io#dom-gpucomputepasstimestampwrites-queryset)
    3. If `timestampWrites`.[endOfPassWriteIndex](https://gpuweb.github.io#dom-gpucomputepasstimestampwrites-endofpasswriteindex)[provided](https://infra.spec.whatwg.org/#map-exists), set `pass`.[\[\[endTimestampWrite\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-endtimestampwrite-slot)[GPU command](https://gpuweb.github.io#gpu-command) with the following steps:
      1. After the pass commands finish executing,
write the [current queue timestamp](https://gpuweb.github.io#abstract-opdef-current-queue-timestamp) into index `timestampWrites`.[endOfPassWriteIndex](https://gpuweb.github.io#dom-gpucomputepasstimestampwrites-endofpasswriteindex)`timestampWrites`.[querySet](https://gpuweb.github.io#dom-gpucomputepasstimestampwrites-queryset)

### 13.4. Buffer Copy Commands

copyBufferToBuffer() has two overloads:

- `copyBufferToBuffer(source, destination, size)`
- Shorthand, equivalent to [copyBufferToBuffer(source, 0, destination, 0, size)](https://gpuweb.github.io#dom-gpucommandencoder-copybuffertobuffer-source-sourceoffset-destination-destinationoffset-size)
- `copyBufferToBuffer(source, sourceOffset, destination, destinationOffset, size)`
- Encode a command into the [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
`this`. **Arguments:**
Arguments for the [GPUCommandEncoder.copyBufferToBuffer(source, sourceOffset, destination, destinationOffset, size)](https://gpuweb.github.io#dom-gpucommandencoder-copybuffertobuffer-source-sourceoffset-destination-destinationoffset-size) method.Parameter Type Nullable Optional Description `source`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ The [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`sourceOffset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✘ Offset in bytes into `source` to begin copying from. `destination`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ The [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`destinationOffset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✘ Offset in bytes into `destination` to place the copied data. `size`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Bytes to copy. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If `size` is `undefined`, set it to `source`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`sourceOffset`.
  3. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `source` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `destination` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `source`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[COPY\_SRC](https://gpuweb.github.io#dom-gpubufferusage-copy_src)
    - `destination`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[COPY\_DST](https://gpuweb.github.io#dom-gpubufferusage-copy_dst)
    - `size` is a multiple of 4.
    - `sourceOffset` is a multiple of 4.
    - `destinationOffset` is a multiple of 4.
    - `source`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`sourceOffset` + `size`).
    - `destination`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`destinationOffset` + `size`).
    - `source` and `destination` are not the same [GPUBuffer](https://gpuweb.github.io#gpubuffer)
  4. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Copy `size` bytes of `source`, beginning at `sourceOffset`, into `destination`,
beginning at `destinationOffset`.

- `clearBuffer(buffer, offset, size)`
- Encode a command into the [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
`this`. **Arguments:**
Arguments for the [GPUCommandEncoder.clearBuffer(buffer, offset, size)](https://gpuweb.github.io#dom-gpucommandencoder-clearbuffer) method.Parameter Type Nullable Optional Description `buffer`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ The [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`offset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Offset in bytes into `buffer` where the sub-region to clear begins. `size`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Size in bytes of the sub-region to clear. Defaults to the size of the buffer minus `offset`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If `size` is missing, set `size` to `max(0,` .`buffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`offset`)
  3. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `buffer` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `buffer`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[COPY\_DST](https://gpuweb.github.io#dom-gpubufferusage-copy_dst)
    - `size` is a multiple of 4.
    - `offset` is a multiple of 4.
    - `buffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`offset` + `size`).
  4. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Set `size` bytes of `buffer` to `0` starting at `offset`.

### 13.5. Texel Copy Commands

- `copyBufferToTexture(source, destination, copySize)`
- Encode a command into the [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[texture subresources](https://gpuweb.github.io#texture-subresources).
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
`this`. **Arguments:**
Arguments for the [GPUCommandEncoder.copyBufferToTexture(source, destination, copySize)](https://gpuweb.github.io#dom-gpucommandencoder-copybuffertotexture) method.Parameter Type Nullable Optional Description `source`
[GPUTexelCopyBufferInfo](https://gpuweb.github.io#gputexelcopybufferinfo)
✘ ✘ Combined with `copySize`, defines the region of the source buffer. `destination`
[GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)
✘ ✘ Combined with `copySize`, defines the region of the destination [texture subresource](https://gpuweb.github.io#texture-subresources). `copySize`
[GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
✘ ✘ **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUOrigin3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuorigin3d-shape)(`destination`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  2. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUExtent3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuextent3d-shape)(`copySize`).
  3. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. Let `aligned` be `true`.
  3. Let `dataLength` be `source`.[buffer](https://gpuweb.github.io#dom-gputexelcopybufferinfo-buffer)[size](https://gpuweb.github.io#dom-gpubuffer-size)
  4. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - [validating GPUTexelCopyBufferInfo](https://gpuweb.github.io#abstract-opdef-validating-gputexelcopybufferinfo)(`source`) returns `true`.
    - `source`.[buffer](https://gpuweb.github.io#dom-gputexelcopybufferinfo-buffer)[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[COPY\_SRC](https://gpuweb.github.io#dom-gpubufferusage-copy_src)
    - [validating texture buffer copy](https://gpuweb.github.io#abstract-opdef-validating-texture-buffer-copy)(`destination`, `source`, `dataLength`, `copySize`, [COPY\_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)`aligned`) returns `true`.
  5. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `blockWidth` be the [texel block width](https://gpuweb.github.io#texel-block-width) of `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  2. Let `blockHeight` be the [texel block height](https://gpuweb.github.io#texel-block-height) of `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  3. Let `dstOrigin` be `destination`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  4. Let `dstBlockOriginX` be (`dstOrigin`.[x](https://gpuweb.github.io#gpuorigin3d-x) ÷ `blockWidth`).
  5. Let `dstBlockOriginY` be (`dstOrigin`.[y](https://gpuweb.github.io#gpuorigin3d-y) ÷ `blockHeight`).
  6. Let `blockColumns` be (`copySize`.[width](https://gpuweb.github.io#gpuextent3d-width) ÷ `blockWidth`).
  7. Let `blockRows` be (`copySize`.[height](https://gpuweb.github.io#gpuextent3d-height) ÷ `blockHeight`).
  8. [Assert](https://infra.spec.whatwg.org/#assert) that `dstBlockOriginX`, `dstBlockOriginY`, `blockColumns`, and `blockRows` are integers.
  9. For each `z` in the range \[0, `copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) − 1\]:
    1. Let `dstSubregion` be [texture copy sub-region](https://gpuweb.github.io#abstract-opdef-texture-copy-sub-region) (`z` + `dstOrigin`.[z](https://gpuweb.github.io#gpuorigin3d-z)) of `destination`.
    2. For each `y` in the range \[0, `blockRows` − 1\]:
      1. For each `x` in the range \[0, `blockColumns` − 1\]:
        1. Let `blockOffset` be the [texel block byte offset](https://gpuweb.github.io#abstract-opdef-texel-block-byte-offset) of `source` for (`x`, `y`, `z`) of `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
        2. Set [texel block](https://gpuweb.github.io#texel-block) (`dstBlockOriginX` + `x`, `dstBlockOriginY` + `y`) of `dstSubregion` to be an [equivalent texel representation](https://gpuweb.github.io#equivalent-texel-representation) to the [texel block](https://gpuweb.github.io#texel-block) described by `source`.[buffer](https://gpuweb.github.io#dom-gputexelcopybufferinfo-buffer)`blockOffset`.
- `copyTextureToBuffer(source, destination, copySize)`
- Encode a command into the [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[texture subresources](https://gpuweb.github.io#texture-subresources) to a sub-region of a [GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
`this`. **Arguments:**
Arguments for the [GPUCommandEncoder.copyTextureToBuffer(source, destination, copySize)](https://gpuweb.github.io#dom-gpucommandencoder-copytexturetobuffer) method.Parameter Type Nullable Optional Description `source`
[GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)
✘ ✘ Combined with `copySize`, defines the region of the source [texture subresources](https://gpuweb.github.io#texture-subresources). `destination`
[GPUTexelCopyBufferInfo](https://gpuweb.github.io#gputexelcopybufferinfo)
✘ ✘ Combined with `copySize`, defines the region of the destination buffer. `copySize`
[GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
✘ ✘ **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUOrigin3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuorigin3d-shape)(`source`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  2. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUExtent3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuextent3d-shape)(`copySize`).
  3. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. Let `aligned` be `true`.
  3. Let `dataLength` be `destination`.[buffer](https://gpuweb.github.io#dom-gputexelcopybufferinfo-buffer)[size](https://gpuweb.github.io#dom-gpubuffer-size)
  4. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - [validating GPUTexelCopyBufferInfo](https://gpuweb.github.io#abstract-opdef-validating-gputexelcopybufferinfo)(`destination`) returns `true`.
    - `destination`.[buffer](https://gpuweb.github.io#dom-gputexelcopybufferinfo-buffer)[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[COPY\_DST](https://gpuweb.github.io#dom-gpubufferusage-copy_dst)
    - [validating texture buffer copy](https://gpuweb.github.io#abstract-opdef-validating-texture-buffer-copy)(`source`, `destination`, `dataLength`, `copySize`, [COPY\_SRC](https://gpuweb.github.io#dom-gputextureusage-copy_src)`aligned`) returns `true`.
    - 
            If device.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
      - `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[format](https://gpuweb.github.io#dom-gputexture-format)[compressed format](https://gpuweb.github.io#compressed-format).
  5. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `blockWidth` be the [texel block width](https://gpuweb.github.io#texel-block-width) of `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  2. Let `blockHeight` be the [texel block height](https://gpuweb.github.io#texel-block-height) of `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  3. Let `srcOrigin` be `source`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  4. Let `srcBlockOriginX` be (`srcOrigin`.[x](https://gpuweb.github.io#gpuorigin3d-x) ÷ `blockWidth`).
  5. Let `srcBlockOriginY` be (`srcOrigin`.[y](https://gpuweb.github.io#gpuorigin3d-y) ÷ `blockHeight`).
  6. Let `blockColumns` be (`copySize`.[width](https://gpuweb.github.io#gpuextent3d-width) ÷ `blockWidth`).
  7. Let `blockRows` be (`copySize`.[height](https://gpuweb.github.io#gpuextent3d-height) ÷ `blockHeight`).
  8. [Assert](https://infra.spec.whatwg.org/#assert) that `srcBlockOriginX`, `srcBlockOriginY`, `blockColumns`, and `blockRows` are integers.
  9. For each `z` in the range \[0, `copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) − 1\]:
    1. Let `srcSubregion` be [texture copy sub-region](https://gpuweb.github.io#abstract-opdef-texture-copy-sub-region) (`z` + `srcOrigin`.[z](https://gpuweb.github.io#gpuorigin3d-z)) of `source`.
    2. For each `y` in the range \[0, `blockRows` − 1\]:
      1. For each `x` in the range \[0, `blockColumns` − 1\]:
        1. Let `blockOffset` be the [texel block byte offset](https://gpuweb.github.io#abstract-opdef-texel-block-byte-offset) of `destination` for (`x`, `y`, `z`) of `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
        2. Set `destination`.[buffer](https://gpuweb.github.io#dom-gputexelcopybufferinfo-buffer)`blockOffset` to be an [equivalent texel representation](https://gpuweb.github.io#equivalent-texel-representation) to [texel block](https://gpuweb.github.io#texel-block) (`srcBlockOriginX` + `x`, `srcBlockOriginY` + `y`) of `srcSubregion`.
- `copyTextureToTexture(source, destination, copySize)`
- Encode a command into the [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[texture subresources](https://gpuweb.github.io#texture-subresources) to another sub-region of one or
multiple continuous [texture subresources](https://gpuweb.github.io#texture-subresources).
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
`this`. **Arguments:**
Arguments for the [GPUCommandEncoder.copyTextureToTexture(source, destination, copySize)](https://gpuweb.github.io#dom-gpucommandencoder-copytexturetotexture) method.Parameter Type Nullable Optional Description `source`
[GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)
✘ ✘ Combined with `copySize`, defines the region of the source [texture subresources](https://gpuweb.github.io#texture-subresources). `destination`
[GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)
✘ ✘ Combined with `copySize`, defines the region of the destination [texture subresources](https://gpuweb.github.io#texture-subresources). `copySize`
[GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
✘ ✘ **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUOrigin3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuorigin3d-shape)(`source`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  2. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUOrigin3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuorigin3d-shape)(`destination`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  3. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUExtent3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuextent3d-shape)(`copySize`).
  4. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - Let `srcTexture` be `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
    - Let `dstTexture` be `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
    - [validating GPUTexelCopyTextureInfo](https://gpuweb.github.io#abstract-opdef-validating-gputexelcopytextureinfo)(`source`, `copySize`) returns `true`.
    - `srcTexture`.[usage](https://gpuweb.github.io#dom-gputexture-usage)[COPY\_SRC](https://gpuweb.github.io#dom-gputextureusage-copy_src)
    - [validating GPUTexelCopyTextureInfo](https://gpuweb.github.io#abstract-opdef-validating-gputexelcopytextureinfo)(`destination`, `copySize`) returns `true`.
    - `dstTexture`.[usage](https://gpuweb.github.io#dom-gputexture-usage)[COPY\_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)
    - `srcTexture`.[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)`dstTexture`.[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)
    - `srcTexture`.[format](https://gpuweb.github.io#dom-gputexture-format)`dstTexture`.[format](https://gpuweb.github.io#dom-gputexture-format)[copy-compatible](https://gpuweb.github.io#copy-compatible).
    - If `srcTexture`.[format](https://gpuweb.github.io#dom-gputexture-format)
    - The [set of subresources for texture copy](https://gpuweb.github.io#abstract-opdef-set-of-subresources-for-texture-copy)(`source`, `copySize`) and
the [set of subresources for texture copy](https://gpuweb.github.io#abstract-opdef-set-of-subresources-for-texture-copy)(`destination`, `copySize`) are disjoint.
    - 
            If device.[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
      - `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[format](https://gpuweb.github.io#dom-gputexture-format)[compressed format](https://gpuweb.github.io#compressed-format).
      - `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[format](https://gpuweb.github.io#dom-gputexture-format)[compressed format](https://gpuweb.github.io#compressed-format).
      - `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)`destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)
  3. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `blockWidth` be the [texel block width](https://gpuweb.github.io#texel-block-width) of `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  2. Let `blockHeight` be the [texel block height](https://gpuweb.github.io#texel-block-height) of `source`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  3. Let `srcOrigin` be `source`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  4. Let `srcBlockOriginX` be (`srcOrigin`.[x](https://gpuweb.github.io#gpuorigin3d-x) ÷ `blockWidth`).
  5. Let `srcBlockOriginY` be (`srcOrigin`.[y](https://gpuweb.github.io#gpuorigin3d-y) ÷ `blockHeight`).
  6. Let `dstOrigin` be `destination`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  7. Let `dstBlockOriginX` be (`dstOrigin`.[x](https://gpuweb.github.io#gpuorigin3d-x) ÷ `blockWidth`).
  8. Let `dstBlockOriginY` be (`dstOrigin`.[y](https://gpuweb.github.io#gpuorigin3d-y) ÷ `blockHeight`).
  9. Let `blockColumns` be (`copySize`.[width](https://gpuweb.github.io#gpuextent3d-width) ÷ `blockWidth`).
  10. Let `blockRows` be (`copySize`.[height](https://gpuweb.github.io#gpuextent3d-height) ÷ `blockHeight`).
  11. [Assert](https://infra.spec.whatwg.org/#assert) that `srcBlockOriginX`, `srcBlockOriginY`, `dstBlockOriginX`, `dstBlockOriginY`, `blockColumns`, and `blockRows` are integers.
  12. For each `z` in the range \[0, `copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) − 1\]:
    1. Let `srcSubregion` be [texture copy sub-region](https://gpuweb.github.io#abstract-opdef-texture-copy-sub-region) (`z` + `srcOrigin`.[z](https://gpuweb.github.io#gpuorigin3d-z)) of `source`.
    2. Let `dstSubregion` be [texture copy sub-region](https://gpuweb.github.io#abstract-opdef-texture-copy-sub-region) (`z` + `dstOrigin`.[z](https://gpuweb.github.io#gpuorigin3d-z)) of `destination`.
    3. For each `y` in the range \[0, `blockRows` − 1\]:
      1. For each `x` in the range \[0, `blockColumns` − 1\]:
        1. Set [texel block](https://gpuweb.github.io#texel-block) (`dstBlockOriginX` + `x`, `dstBlockOriginY` + `y`) of `dstSubregion` to be an [equivalent texel representation](https://gpuweb.github.io#equivalent-texel-representation) to [texel block](https://gpuweb.github.io#texel-block) (`srcBlockOriginX` + `x`, `srcBlockOriginY` + `y`) of `srcSubregion`.

### 13.6. Queries

- `resolveQuerySet(querySet, firstQuery, queryCount, destination, destinationOffset)`
- Resolves query results from a [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)[GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
**Arguments:**
Arguments for the [GPUCommandEncoder.resolveQuerySet(querySet, firstQuery, queryCount, destination, destinationOffset)](https://gpuweb.github.io#dom-gpucommandencoder-resolvequeryset) method.Parameter Type Nullable Optional Description `querySet` [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
✘ ✘ `firstQuery` [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✘ `queryCount` [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✘ `destination` [GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ `destinationOffset` [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✘ **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `querySet` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `destination` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `destination`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[QUERY\_RESOLVE](https://gpuweb.github.io#dom-gpubufferusage-query_resolve)
    - `firstQuery` \< the number of queries in `querySet`.
    - (`firstQuery` + `queryCount`) ≤ the number of queries in `querySet`.
    - `destinationOffset` is a multiple of 256.
    - `destinationOffset` + 8 × `queryCount` ≤ `destination`.[size](https://gpuweb.github.io#dom-gpubuffer-size)
  3. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `queryIndex` be `firstQuery`.
  2. Let `offset` be `destinationOffset`.
  3. While `queryIndex` \< `firstQuery` + `queryCount`:
    1. Set 8 bytes of `destination`, beginning at `offset`, to be the value of `querySet` at `queryIndex`.
    2. Set `queryIndex` to be `queryIndex` + 1.
    3. Set `offset` to be `offset` + 8.

### 13.7. Finalization

A [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)[GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[finish()](https://gpuweb.github.io#dom-gpucommandencoder-finish)[finish()](https://gpuweb.github.io#dom-gpucommandencoder-finish)

- `finish(descriptor)`
- Completes recording of the commands sequence and returns a corresponding [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)
**Called on:** [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
`this`. **Arguments:**
Arguments for the [GPUCommandEncoder.finish(descriptor)](https://gpuweb.github.io#dom-gpucommandencoder-finish) method.Parameter Type Nullable Optional Description `descriptor` [GPUCommandBufferDescriptor](https://gpuweb.github.io#dictdef-gpucommandbufferdescriptor)
✘ ✔ **Returns:** [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `commandBuffer` be a new [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)
  2. Issue the `finish steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  3. Return `commandBuffer`.
[Device timeline](https://gpuweb.github.io#device-timeline) `finish steps`:
  1. Let `validationSucceeded` be `true` if all of the following requirements are met, and `false` otherwise.
    - `this` must be [valid](https://gpuweb.github.io#abstract-opdef-valid).
    - `this`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[open](https://gpuweb.github.io#encoder-state-open)".
    - `this`.[\[\[debug\_group\_stack\]\]](https://gpuweb.github.io#dom-gpudebugcommandsmixin-debug_group_stack-slot)[be empty](https://infra.spec.whatwg.org/#list-is-empty).
  2. Set `this`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[ended](https://gpuweb.github.io#encoder-state-ended)".
  3. If `validationSucceeded` is `false`, then:
    1. [Generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).
    2. Return an [invalidated](https://gpuweb.github.io#abstract-opdef-invalidate) [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)
  4. Set `commandBuffer`.[\[\[command\_list\]\]](https://gpuweb.github.io#dom-gpucommandbuffer-command_list-slot)`this`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)
  5. Set `commandBuffer`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandbuffer-used_bind_groups-slot)`this`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-used_bind_groups-slot)

## 14. Programmable Passes

interface mixin GPUBindingCommandsMixin undefined setBindGroup GPUIndex32 index GPUBindGroup bindGroup optional sequence GPUBufferDynamicOffset dynamicOffsets undefined setBindGroup GPUIndex32 index GPUBindGroup bindGroup AllowShared Uint32Array dynamicOffsetsData GPUSize64 dynamicOffsetsDataStart GPUSize32 dynamicOffsetsDataLength undefined setImmediates GPUSize32 rangeOffset AllowSharedBufferSource data optional GPUSize64 dataOffset optional GPUSize64 dataSize 

[GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)[GPUCommandsMixin](https://gpuweb.github.io#gpucommandsmixin)

[GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[bind_groups]]`, of type [ordered map](https://infra.spec.whatwg.org/#ordered-map)\<[GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)
[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
- The current [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
- `[[dynamic_offsets]]`, of type [ordered map](https://infra.spec.whatwg.org/#ordered-map)\<[GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)
[list](https://infra.spec.whatwg.org/#list)\<[GPUBufferDynamicOffset](https://gpuweb.github.io#typedefdef-gpubufferdynamicoffset)
- The current dynamic offsets for each [\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)
- `[[immediate_data]]`, of type [byte sequence](https://infra.spec.whatwg.org/#byte-sequence)
- The current immediate data bytes.
Initially a [byte sequence](https://infra.spec.whatwg.org/#byte-sequence) of length equal to the device’s [maxImmediateSize](https://gpuweb.github.io#dom-supported-limits-maximmediatesize)[setImmediates()](https://gpuweb.github.io#dom-gpubindingcommandsmixin-setimmediates)
- `[[immediate_slots_set]]`, of type [list](https://infra.spec.whatwg.org/#list)\<[boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- Tracks which 32-bit word slots of immediate data have been set.
Initially a [list](https://infra.spec.whatwg.org/#list) of length equal to the device’s [maxImmediateSize](https://gpuweb.github.io#dom-supported-limits-maximmediatesize)`false`.
Each entry corresponds to a 4-byte slot.

### 14.1. Bind Groups

setBindGroup() has two overloads:

- `setBindGroup(index, bindGroup, dynamicOffsets)`
- Sets the current [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
**Called on:** [GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)
**Arguments:**
  - `index`
[GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)
  - The index to set the bind group at.
  - `bindGroup`
[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
  - Bind group to use for subsequent render or compute commands.
  - `dynamicOffsets`
[sequence](https://webidl.spec.whatwg.org/#idl-sequence)\<[GPUBufferDynamicOffset](https://gpuweb.github.io#typedefdef-gpubufferdynamicoffset)
`[]`
  - Array containing buffer offsets in bytes for each entry in
`bindGroup` marked as [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)[note](https://gpuweb.github.io#dynamicOffsetOrder) for additional details.
**Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. Let `dynamicOffsetCount` be 0 if `bindGroup` is `null`, or `bindGroup`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpubindgroup-layout-slot)[\[\[dynamicOffsetCount\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-dynamicoffsetcount-slot)
  3. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `index` must be \< `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxBindGroups](https://gpuweb.github.io#dom-supported-limits-maxbindgroups)
    - `dynamicOffsets`.[size](https://infra.spec.whatwg.org/#list-size) must equal `dynamicOffsetCount`.
  4. If `bindGroup` is `null`:
    1. [Remove](https://infra.spec.whatwg.org/#map-remove) `this`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)`index`\].
    2. [Remove](https://infra.spec.whatwg.org/#map-remove) `this`.[\[\[dynamic\_offsets\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-dynamic_offsets-slot)`index`\].
Otherwise: 
    1. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
      - `bindGroup` must be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
      - [For each dynamic binding](https://gpuweb.github.io#abstract-opdef-iterate-over-each-dynamic-binding-offset) (`bufferBinding`, `bufferLayout`, `dynamicOffsetIndex`) in `bindGroup`:
        - `bufferBinding`.[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)`dynamicOffsets`\[`dynamicOffsetIndex`\] + `bufferLayout`.[minBindingSize](https://gpuweb.github.io#dom-gpubufferbindinglayout-minbindingsize)`bufferBinding`.[buffer](https://gpuweb.github.io#dom-gpubufferbinding-buffer)[size](https://gpuweb.github.io#dom-gpubuffer-size)
        - If `bufferLayout`.[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)["uniform"](https://gpuweb.github.io#dom-gpubufferbindingtype-uniform)
          - `dynamicOffset` must be a multiple of [minUniformBufferOffsetAlignment](https://gpuweb.github.io#dom-supported-limits-minuniformbufferoffsetalignment)
        - If `bufferLayout`.[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)["read-only-storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-read-only-storage)
          - `dynamicOffset` must be a multiple of [minStorageBufferOffsetAlignment](https://gpuweb.github.io#dom-supported-limits-minstoragebufferoffsetalignment)
    2. Set `this`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)`index`\] to be `bindGroup`.
    3. Set `this`.[\[\[dynamic\_offsets\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-dynamic_offsets-slot)`index`\] to be a copy of `dynamicOffsets`.
    4. [Append](https://infra.spec.whatwg.org/#set-append) `bindGroup` to `this`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-used_bind_groups-slot)
    5. If `this` is a [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
      1. For each `bindGroup` in `this`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)[merge](https://gpuweb.github.io#abstract-opdef-usage-scope-merge) `bindGroup`.[\[\[usedResources\]\]](https://gpuweb.github.io#dom-gpubindgroup-usedresources-slot)`this`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)
- `setBindGroup(index, bindGroup, dynamicOffsetsData, dynamicOffsetsDataStart, dynamicOffsetsDataLength)`
- Sets the current [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)[Uint32Array](https://webidl.spec.whatwg.org/#idl-Uint32Array)
**Called on:** [GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)
`this`. **Arguments:**
Arguments for the [GPUBindingCommandsMixin.setBindGroup(index, bindGroup, dynamicOffsetsData, dynamicOffsetsDataStart, dynamicOffsetsDataLength)](https://gpuweb.github.io#dom-gpubindingcommandsmixin-setbindgroup-index-bindgroup-dynamicoffsetsdata-dynamicoffsetsdatastart-dynamicoffsetsdatalength) method.Parameter Type Nullable Optional Description `index`
[GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)
✘ ✘ The index to set the bind group at. `bindGroup`
[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)?
✔ ✘ Bind group to use for subsequent render or compute commands. `dynamicOffsetsData`
[Uint32Array](https://webidl.spec.whatwg.org/#idl-Uint32Array)
✘ ✘ Array containing buffer offsets in bytes for each entry in `bindGroup` marked as [buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)
[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)
[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)
[note](https://gpuweb.github.io#dynamicOffsetOrder) for additional details. `dynamicOffsetsDataStart`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✘ Offset in elements into `dynamicOffsetsData` where the
buffer offset data begins. `dynamicOffsetsDataLength`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✘ Number of buffer offsets to read from `dynamicOffsetsData`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If any of the following requirements are unmet, throw a [RangeError](https://webidl.spec.whatwg.org/#exceptiondef-rangeerror)
    - `dynamicOffsetsDataStart` must be ≥ 0.
    - `dynamicOffsetsDataStart` + `dynamicOffsetsDataLength` must be ≤ `dynamicOffsetsData`.`length`.
  2. Let `dynamicOffsets` be a [list](https://infra.spec.whatwg.org/#list) containing the range, starting at index `dynamicOffsetsDataStart`, of `dynamicOffsetsDataLength` elements of [a copy of](https://webidl.spec.whatwg.org/#dfn-get-buffer-source-copy) `dynamicOffsetsData`.
  3. Call `this`.[setBindGroup](https://gpuweb.github.io#dom-gpubindingcommandsmixin-setbindgroup)`index`, `bindGroup`, `dynamicOffsets`).


[GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)
This means that if `dynamic bindings` is the list of each [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)`true`, sorted by
    [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)[binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)`dynamic offset[i]`, as supplied to
    [setBindGroup()](https://gpuweb.github.io#gpubindingcommandsmixin-setbindgroup), will correspond to `dynamic bindings[i]`.

[For a](https://gpuweb.github.io#example-3dbbcd16)

[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)
// Note the bindings are listed out-of-order in this array, but it // doesn't matter because they will be sorted by binding index. let layout= gpuDevice. createBindGroupLayout({ entries: \[{ binding: 1 , buffer: {}, }, { binding: 2 , buffer: { dynamicOffset: true }, }, { binding: 0 , buffer: { dynamicOffset: true }, }\] }); 

Used by a [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)

// Like above, the array order doesn't matter here. // It doesn't even need to match the order used in the layout. let bindGroup= gpuDevice. createBindGroup({ layout: layout, entries: \[{ binding: 1 , resource: { buffer: bufferA, offset: 256 }, }, { binding: 2 , resource: { buffer: bufferB, offset: 512 }, }, { binding: 0 , resource: { buffer: bufferC}, }\] }); 

And bound with the following call:

pass. setBindGroup( 0 , bindGroup, \[ 1024 , 2048 \]); 

The following buffer offsets will be applied:

| Binding | Buffer | Offset | 
|---|---|---|
| 0 | bufferC | 1024 (Dynamic) | 
| 1 | bufferA | 256 (Static) | 
| 2 | bufferB | 2560 (Static + Dynamic) | 

[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
`bindGroup`with a given list of

`steps`to be executed for each dynamic offset, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. Let `dynamicOffsetIndex` be `0`.
2. Let `layout` be `bindGroup`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpubindgroup-layout-slot)
3. For each [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)`entry` in `bindGroup`.[\[\[entries\]\]](https://gpuweb.github.io#dom-gpubindgroup-entries-slot)`entry`.[binding](https://gpuweb.github.io#dom-gpubindgroupentry-binding)
  1. Let `bindingDescriptor` be the [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`layout`.[\[\[entryMap\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-entrymap-slot)`entry`.[binding](https://gpuweb.github.io#dom-gpubindgroupentry-binding)
  2. If `bindingDescriptor`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)`true`:
    1. Let `bufferBinding` be [get as buffer binding](https://gpuweb.github.io#abstract-opdef-get-as-buffer-binding)(`entry`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)
    2. Let `bufferLayout` be `bindingDescriptor`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)
    3. Call `steps` with `bufferBinding`, `bufferLayout`, and `dynamicOffsetIndex`.
    4. Let `dynamicOffsetIndex` be `dynamicOffsetIndex` + `1`

**Arguments:**

- [GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)
`encoder`
- Encoder whose bind groups are being validated.
- [GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)
`pipeline`
- Pipeline to validate `encoder`s bind groups are compatible with.

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. If any of the following conditions are unsatisfied, return `false`:
  - Let `bindGroupLayouts` be `pipeline`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpupipelinebase-layout-slot)[\[\[bindGroupLayouts\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)
  - `pipeline` must not be `null`.
  - All bind groups used by the pipeline must be set and compatible with the pipeline layout, determined as follows: For each pair of ([GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)`index`, [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)`bindGroupLayout`) in `bindGroupLayouts`:
    - If `bindGroupLayout` is `null`, [continue](https://infra.spec.whatwg.org/#iteration-continue).
    - Let `bindGroup` be `encoder`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)`index`\].
    - Let `dynamicOffsets` be `encoder`.[\[\[dynamic\_offsets\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-dynamic_offsets-slot)`index`\].
    - `bindGroup` must not be `null`.
    - `bindGroup`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpubindgroup-layout-slot)[group-equivalent](https://gpuweb.github.io#group-equivalent) with `bindGroupLayout`.
    - Let `dynamicOffsetIndex` be 0.
    - For each [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)`bindGroupEntry` in `bindGroup`.[\[\[entries\]\]](https://gpuweb.github.io#dom-gpubindgroup-entries-slot)`bindGroupEntry`.[binding](https://gpuweb.github.io#dom-gpubindgroupentry-binding)
      - Let `bindGroupLayoutEntry` be `bindGroup`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpubindgroup-layout-slot)[\[\[entryMap\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-entrymap-slot)`bindGroupEntry`.[binding](https://gpuweb.github.io#dom-gpubindgroupentry-binding)
      - If `bindGroupLayoutEntry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[provided](https://infra.spec.whatwg.org/#map-exists), **continue**.
      - Let `bound` be [get as buffer binding](https://gpuweb.github.io#abstract-opdef-get-as-buffer-binding)(`bindGroupEntry`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)
      - If `bindGroupLayoutEntry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[hasDynamicOffset](https://gpuweb.github.io#dom-gpubufferbindinglayout-hasdynamicoffset)
        - Increment `bound`.[offset](https://gpuweb.github.io#dom-gpubufferbinding-offset)`dynamicOffsets`\[`dynamicOffsetIndex`\].
        - Increment `dynamicOffsetIndex` by 1.
      - If `bindGroupEntry`.[\[\[prevalidatedSize\]\]](https://gpuweb.github.io#dom-gpubindgroupentry-prevalidatedsize-slot)`false`:
        - [effective buffer binding size](https://gpuweb.github.io#abstract-opdef-effective-buffer-binding-size)(`bound`) must be ≥ [minimum buffer binding size](https://gpuweb.github.io#minimum-buffer-binding-size) of the binding variable in `pipeline`’s shader that corresponds to `bindGroupEntry`.
  - [Encoder bind groups alias a writable resource](https://gpuweb.github.io#abstract-opdef-encoder-bind-groups-alias-a-writable-resource)(`encoder`, `pipeline`) must be `false`.
  - 
         If`encoder`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)
[contain](https://infra.spec.whatwg.org/#list-contain) ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
    - All bindings referring to the same [GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureView](https://gpuweb.github.io#gputextureview)
For each pair of ([GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)`index1`, [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)`bindGroupLayout1`) in `bindGroupLayouts`:
      - If `bindGroupLayout1` is `null`, [continue](https://infra.spec.whatwg.org/#iteration-continue).
      - Let `bindGroup1` be `encoder`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)`index1`\].
      - For each [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)`bindGroupEntry1` in `bindGroup1`.[\[\[entries\]\]](https://gpuweb.github.io#dom-gpubindgroup-entries-slot)
        - If `bindGroupEntry1`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)[GPUTextureView](https://gpuweb.github.io#gputextureview)[continue](https://infra.spec.whatwg.org/#iteration-continue).
        - Let `descriptor1` be `bindGroupEntry1`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)
        - For each pair of ([GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)`index2`, [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)`bindGroupLayout2`) in `bindGroupLayouts`:
          - If `bindGroupLayout2` is `null`, [continue](https://infra.spec.whatwg.org/#iteration-continue).
          - Let `bindGroup2` be `encoder`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)`index2`\].
          - For each [GPUBindGroupEntry](https://gpuweb.github.io#dictdef-gpubindgroupentry)`bindGroupEntry2` in `bindGroup2`.[\[\[entries\]\]](https://gpuweb.github.io#dom-gpubindgroup-entries-slot)
            - If `bindGroupEntry2`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)[GPUTextureView](https://gpuweb.github.io#gputextureview)[continue](https://infra.spec.whatwg.org/#iteration-continue).
            - If `bindGroupEntry1`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)`bindGroupEntry2`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)[continue](https://infra.spec.whatwg.org/#iteration-continue).
            - Let `descriptor2` be `bindGroupEntry2`.[resource](https://gpuweb.github.io#dom-gpubindgroupentry-resource)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)
            - `descriptor2`.[baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)`descriptor1`.[baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)
            - `descriptor2`.[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)`descriptor1`.[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)
            - `descriptor2`.[aspect](https://gpuweb.github.io#dom-gputextureviewdescriptor-aspect)`descriptor1`.[aspect](https://gpuweb.github.io#dom-gputextureviewdescriptor-aspect)
            - `descriptor2`.[swizzle](https://gpuweb.github.io#dom-gputextureviewdescriptor-swizzle)`descriptor1`.[swizzle](https://gpuweb.github.io#dom-gputextureviewdescriptor-swizzle)

Otherwise return `true`.

**Arguments:**

- [GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)
`encoder`
- Encoder whose immediate data is being validated.
- [GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)
`pipeline`
- Pipeline to validate `encoder`’s immediate data is compatible with.

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. If any of the following conditions are unsatisfied, return `false`:
  - `pipeline` must not be `null`.
  - For each immediate data variable that is [statically used](https://gpuweb.github.io#statically-used) by any entry point in `pipeline`:
    - Let `T` be the [store type](https://gpuweb.github.io/gpuweb/wgsl/#store-type) of the immediate data variable.
    - Let `accessibleSlots` be [AccessibleSlots](https://gpuweb.github.io/gpuweb/wgsl/#accessible-slots)(`T`) as defined in [\[WGSL\]](https://gpuweb.github.io#biblio-wgsl).
    - For each `slotIndex` in `accessibleSlots`:
      - `encoder`.[\[\[immediate\_slots\_set\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-immediate_slots_set-slot)`slotIndex`\] must be `true`.
Otherwise return `true`.

`encoder`,

`pipeline`) if any writable buffer binding range overlaps with any other binding range of the same buffer, or any writable texture binding overlaps in

[texture subresources](https://gpuweb.github.io#texture-subresources)with any other texture binding (which may use the same or a different

[GPUTextureView](https://gpuweb.github.io#gputextureview)
Note: This algorithm limits the use of the [usage scope storage exception](https://gpuweb.github.io#usage-scope-storage-exception).

**Arguments:**

- [GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)
`encoder`
- Encoder whose bind groups are being validated.
- [GPUPipelineBase](https://gpuweb.github.io#gpupipelinebase)
`pipeline`
- Pipeline to validate `encoder`s bind groups are compatible with.

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. For each `stage` in \[[VERTEX](https://gpuweb.github.io#dom-gpushaderstage-vertex)[FRAGMENT](https://gpuweb.github.io#dom-gpushaderstage-fragment)[COMPUTE](https://gpuweb.github.io#dom-gpushaderstage-compute)
  1. Let `bufferBindings` be a [list](https://infra.spec.whatwg.org/#list) of ([GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)[boolean](https://webidl.spec.whatwg.org/#idl-boolean)
  2. Let `textureViews` be a [list](https://infra.spec.whatwg.org/#list) of ([GPUTextureView](https://gpuweb.github.io#gputextureview)[boolean](https://webidl.spec.whatwg.org/#idl-boolean)
  3. For each pair of ([GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)`bindGroupIndex`, [GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)`bindGroupLayout`) in `pipeline`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpupipelinebase-layout-slot)[\[\[bindGroupLayouts\]\]](https://gpuweb.github.io#dom-gpupipelinelayout-bindgrouplayouts-slot)
    1. Let `bindGroup` be `encoder`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)`bindGroupIndex`\].
    2. Let `bindGroupLayoutEntries` be `bindGroupLayout`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpubindgrouplayout-descriptor-slot)[entries](https://gpuweb.github.io#dom-gpubindgrouplayoutdescriptor-entries)
    3. Let `bufferRanges` be the [bound buffer ranges](https://gpuweb.github.io#gpubindgroup-bound-buffer-ranges) of `bindGroup`,
given dynamic offsets `encoder`.[\[\[dynamic\_offsets\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-dynamic_offsets-slot)`bindGroupIndex`\]
    4. For each ([GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`bindGroupLayoutEntry`, [GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)`resource`) in `bufferRanges`, in which `bindGroupLayoutEntry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)`stage`:
      1. Let `resourceWritable` be (`bindGroupLayoutEntry`.[buffer](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-buffer)[type](https://gpuweb.github.io#dom-gpubufferbindinglayout-type)["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)
      2. For each pair ([GPUBufferBinding](https://gpuweb.github.io#dictdef-gpubufferbinding)`pastResource`, [boolean](https://webidl.spec.whatwg.org/#idl-boolean)`pastResourceWritable`) in `bufferBindings`:
        1. If (`resourceWritable` or `pastResourceWritable`) is true, and `pastResource` and `resource` are [buffer-binding-aliasing](https://gpuweb.github.io#buffer-binding-aliasing), return `true`.
      3. [Append](https://infra.spec.whatwg.org/#list-append) (`resource`, `resourceWritable`) to `bufferBindings`.
    5. For each [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)`bindGroupLayoutEntry` in `bindGroupLayoutEntries`, and corresponding [GPUTextureView](https://gpuweb.github.io#gputextureview)`resource` in `bindGroup`, in which `bindGroupLayoutEntry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)`stage`:
      1. If `bindGroupLayoutEntry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[provided](https://infra.spec.whatwg.org/#map-exists), **continue**.
      2. Let `resourceWritable` be whether `bindGroupLayoutEntry`.[storageTexture](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-storagetexture)[access](https://gpuweb.github.io#dom-gpustoragetexturebindinglayout-access)
      3. For each pair ([GPUTextureView](https://gpuweb.github.io#gputextureview)`pastResource`, [boolean](https://webidl.spec.whatwg.org/#idl-boolean)`pastResourceWritable`) in `textureViews`,
        1. If (`resourceWritable` or `pastResourceWritable`) is true, and `pastResource` and `resource` is [texture-view-aliasing](https://gpuweb.github.io#texture-view-aliasing), return `true`.
      4. [Append](https://infra.spec.whatwg.org/#list-append) (`resource`, `resourceWritable`) to `textureViews`.
2. Return `false`.

Note: Implementations are strongly encouraged to optimize this algorithm.

### 14.2. Immediate Data

- `setImmediates(rangeOffset, data, dataOffset, dataSize)`
- Sets immediate data for subsequent render or compute commands. **Called on:** [GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)
`this`. **Arguments:**
Arguments for the [GPUBindingCommandsMixin.setImmediates(rangeOffset, data, dataOffset, dataSize)](https://gpuweb.github.io#dom-gpubindingcommandsmixin-setimmediates) method.Parameter Type Nullable Optional Description `rangeOffset`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✘ Offset in bytes into the immediate data range to begin writing at. `data`
[AllowSharedBufferSource](https://webidl.spec.whatwg.org/#AllowSharedBufferSource)
✘ ✘ Data to write into the immediate data range. `dataOffset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Offset into `data` to begin writing from. Given in elements if `data` is a [TypedArray](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-typedarray-objects)
`dataSize`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Size of content to write from `data`. Given in elements if `data` is a [TypedArray](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-typedarray-objects)
**Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If `data` is an [ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[DataView](https://webidl.spec.whatwg.org/#idl-DataView)`data` is a TypedArray; let the element type be the type of the TypedArray.
  2. Let `dataElementCount` be the size of `data`, in elements.
  3. If `dataSize` is missing,
let `contentsSize` be `dataElementCount` − `dataOffset`.
Otherwise, let `contentsSize` be `dataSize`.
  4. If any of the following conditions are unsatisfied,
throw an [OperationError](https://webidl.spec.whatwg.org/#operationerror)
    - `contentsSize` ≥ 0.
    - `dataOffset` + `contentsSize` ≤ `dataElementCount`.
    - `contentsSize`, converted to bytes, is a multiple of 4.
  5. Let `dataContents` be [a copy of the bytes held by the buffer source](https://webidl.spec.whatwg.org/#dfn-get-buffer-source-copy) `data`.
  6. Let `contents` be the `contentsSize` elements of `dataContents` starting at
an offset of `dataOffset` elements.
  7. Let `contentsBytes` be `contentsSize` converted to bytes.
  8. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `rangeOffset` is a multiple of 4.
    - `rangeOffset` + `contentsBytes` ≤ `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxImmediateSize](https://gpuweb.github.io#dom-supported-limits-maximmediatesize)
  3. For each byte `b` at index `i` in `contents`:
    1. Let `byteIndex` be `rangeOffset` + `i`.
    2. Let `slotIndex` be floor(`byteIndex` ÷ 4).
    3. Set `this`.[\[\[immediate\_data\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-immediate_data-slot)`byteIndex`\] to `b`.
    4. Set `this`.[\[\[immediate\_slots\_set\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-immediate_slots_set-slot)`slotIndex`\] to true.

## 15. Debug Markers

`GPUDebugCommandsMixin` provides methods to apply debug labels to groups
of commands or insert a single label into the command sequence.

Debug groups can be nested to create a hierarchy of labeled commands, and must be well-balanced.

Like [object labels](https://gpuweb.github.io#dom-gpuobjectbase-label)

interface mixin GPUDebugCommandsMixin undefined pushDebugGroup USVString groupLabel undefined popDebugGroup undefined insertDebugMarker USVString markerLabel 

[GPUDebugCommandsMixin](https://gpuweb.github.io#gpudebugcommandsmixin)[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)[GPUCommandsMixin](https://gpuweb.github.io#gpucommandsmixin)

[GPUDebugCommandsMixin](https://gpuweb.github.io#gpudebugcommandsmixin)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

[GPUDebugCommandsMixin](https://gpuweb.github.io#gpudebugcommandsmixin)

- `pushDebugGroup(groupLabel)`
- Begins a labeled debug group containing subsequent commands. **Called on:** [GPUDebugCommandsMixin](https://gpuweb.github.io#gpudebugcommandsmixin)
`this`. **Arguments:**
Arguments for the [GPUDebugCommandsMixin.pushDebugGroup(groupLabel)](https://gpuweb.github.io#dom-gpudebugcommandsmixin-pushdebuggroup) method.Parameter Type Nullable Optional Description `groupLabel`
[USVString](https://webidl.spec.whatwg.org/#idl-USVString)
✘ ✘ The label for the command group. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. [Push](https://infra.spec.whatwg.org/#stack-push) `groupLabel` onto `this`.[\[\[debug\_group\_stack\]\]](https://gpuweb.github.io#dom-gpudebugcommandsmixin-debug_group_stack-slot)
- `popDebugGroup()`
- Ends the labeled debug group most recently started by [pushDebugGroup()](https://gpuweb.github.io#dom-gpudebugcommandsmixin-pushdebuggroup)
**Called on:** [GPUDebugCommandsMixin](https://gpuweb.github.io#gpudebugcommandsmixin)
`this`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `this`.[\[\[debug\_group\_stack\]\]](https://gpuweb.github.io#dom-gpudebugcommandsmixin-debug_group_stack-slot)[be empty](https://infra.spec.whatwg.org/#list-is-empty).
  3. [Pop](https://infra.spec.whatwg.org/#stack-pop) an entry off of `this`.[\[\[debug\_group\_stack\]\]](https://gpuweb.github.io#dom-gpudebugcommandsmixin-debug_group_stack-slot)
- `insertDebugMarker(markerLabel)`
- Marks a point in a stream of commands with a label. **Called on:** [GPUDebugCommandsMixin](https://gpuweb.github.io#gpudebugcommandsmixin)
**Arguments:**
Arguments for the [GPUDebugCommandsMixin.insertDebugMarker(markerLabel)](https://gpuweb.github.io#dom-gpudebugcommandsmixin-insertdebugmarker) method.Parameter Type Nullable Optional Description `markerLabel` [USVString](https://webidl.spec.whatwg.org/#idl-USVString)
✘ ✘ The label to insert. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.

## 16. Compute Passes

### 16.1. `GPUComputePassEncoder`

\[Exposed Window ,Worker ),SecureContext interface GPUComputePassEncoder undefined setPipeline GPUComputePipeline pipeline undefined dispatchWorkgroups GPUSize32 workgroupCountX optional GPUSize32 workgroupCountY optional GPUSize32 workgroupCountZ undefined dispatchWorkgroupsIndirect GPUBuffer indirectBuffer GPUSize64 indirectOffset undefined end GPUComputePassEncoder includes GPUObjectBase GPUComputePassEncoder includes GPUCommandsMixin GPUComputePassEncoder includes GPUDebugCommandsMixin GPUComputePassEncoder includes GPUBindingCommandsMixin 

[GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[command_encoder]]`, of type [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
- The [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
- `[[endTimestampWrite]]`, of type [GPU command](https://gpuweb.github.io#gpu-command)?, readonly, defaulting to `null`
- [GPU command](https://gpuweb.github.io#gpu-command), if any, writing a timestamp when the pass ends.
- `[[pipeline]]`, of type [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
`null`
- The current [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)

#### 16.1.1. Compute Pass Encoder Creation

dictionary GPUComputePassTimestampWrites required GPUQuerySet querySet GPUSize32 beginningOfPassWriteIndex GPUSize32 endOfPassWriteIndex 

- `querySet`,  of type [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
- The [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)["timestamp"](https://gpuweb.github.io#dom-gpuquerytype-timestamp)
- `beginningOfPassWriteIndex`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- If defined, indicates the query index in [querySet](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-queryset)
- `endOfPassWriteIndex`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- If defined, indicates the query index in [querySet](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-queryset)

Note: Timestamp query values are written in nanoseconds, but how the value is determined is
[implementation-defined](https://infra.spec.whatwg.org/#implementation-defined). See [§ 20.4 Timestamp Query](https://gpuweb.github.io#timestamp) for details.

dictionary GPUComputePassDescriptor GPUObjectDescriptorBase GPUComputePassTimestampWrites timestampWrites 

- `timestampWrites`,  of type [GPUComputePassTimestampWrites](https://gpuweb.github.io#dictdef-gpucomputepasstimestampwrites)
- Defines which timestamp values will be written for this pass, and where to write them to.

#### 16.1.2. Dispatch

- `setPipeline(pipeline)`
- Sets the current [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
**Called on:** [GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)
**Arguments:**
Arguments for the [GPUComputePassEncoder.setPipeline(pipeline)](https://gpuweb.github.io#dom-gpucomputepassencoder-setpipeline) method.Parameter Type Nullable Optional Description `pipeline`
[GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
✘ ✘ The compute pipeline to use for subsequent dispatch commands. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `pipeline` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
  3. Set `this`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)`pipeline`.
- `dispatchWorkgroups(workgroupCountX, workgroupCountY, workgroupCountZ)`
- Dispatch work to be performed with the current [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[§ 23.1 Computing](https://gpuweb.github.io#computing-operations) for the detailed specification.
**Called on:** [GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)
**Arguments:**
Arguments for the [GPUComputePassEncoder.dispatchWorkgroups(workgroupCountX, workgroupCountY, workgroupCountZ)](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups) method.Parameter Type Nullable Optional Description `workgroupCountX`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✘ X dimension of the grid of workgroups to dispatch. `workgroupCountY`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✔ Y dimension of the grid of workgroups to dispatch. `workgroupCountZ`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✔ Z dimension of the grid of workgroups to dispatch. NOTE:The`x`, `y`, and `z` values passed to [dispatchWorkgroups()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups)
[dispatchWorkgroupsIndirect()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroupsindirect)
*workgroups* to dispatch for each dimension, *not* the number of shader invocations
            to perform across each dimension. This matches the behavior of modern native GPU
            APIs, but differs from the behavior of OpenCL. This means that if a [GPUShaderModule](https://gpuweb.github.io#gpushadermodule)`@workgroup_size(4, 4)`, and work is dispatched to it with the call `computePass.dispatchWorkgroups(8, 8);` the entry point will be invoked 1024 times
            total: Dispatching a 4x4 workgroup 8 times along both the X and Y axes.
            (`4*4*8*8=1024`)
**Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. Let `usageScope` be an empty [usage scope](https://gpuweb.github.io#usage-scope).
  3. For each `bindGroup` in `this`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)[merge](https://gpuweb.github.io#abstract-opdef-usage-scope-merge) `bindGroup`.[\[\[usedResources\]\]](https://gpuweb.github.io#dom-gpubindgroup-usedresources-slot)`this`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)
  4. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `usageScope` must satisfy [usage scope validation](https://gpuweb.github.io#usage-scope-validation).
    - [Validate encoder bind groups](https://gpuweb.github.io#abstract-opdef-validate-encoder-bind-groups)(`this`, `this`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)`true`.
    - [Validate immediate data](https://gpuweb.github.io#abstract-opdef-validate-immediate-data)(`this`, `this`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)`true`.
    - all of `workgroupCountX`, `workgroupCountY` and `workgroupCountZ` are ≤ `this`.device.limits.[maxComputeWorkgroupsPerDimension](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsperdimension)
    - let `workgroupSize` be the computed workgroup size for `bindingState`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)
    - the entry point uses the [workgroup\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-workgroup_index) built-in value and `workgroupCountX` × `workgroupCountY` × `workgroupCountZ` is out of range of an unsigned 32-bit integer.
    - the entry point uses the [global\_invocation\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-global_invocation_index) built-in value and `workgroupCountX` × `workgroupCountY` × `workgroupCountZ` × `workgroupSize` is out of range of an unsigned 32-bit integer.
  5. Let `bindingState` be a snapshot of `this`’s current state.
  6. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline).
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Execute a grid of workgroups with dimensions \[`workgroupCountX`, `workgroupCountY`, `workgroupCountZ`\] with `bindingState`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)`bindingState`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)
- `dispatchWorkgroupsIndirect(indirectBuffer, indirectOffset)`
- Dispatch work to be performed with the current [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[§ 23.1 Computing](https://gpuweb.github.io#computing-operations) for the detailed specification.
The indirect dispatch parameters encoded in the buffer must be a tightly
packed block of **three 32-bit unsigned integer values (12 bytes total)**,
given in the same order as the arguments for [dispatchWorkgroups()](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroups)
let dispatchIndirectParameters= new Uint32Array( 3 ); dispatchIndirectParameters\[ 0 \] = workgroupCountX; dispatchIndirectParameters\[ 1 \] = workgroupCountY; dispatchIndirectParameters\[ 2 \] = workgroupCountZ; **Called on:** [GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)
**Arguments:**
Arguments for the [GPUComputePassEncoder.dispatchWorkgroupsIndirect(indirectBuffer, indirectOffset)](https://gpuweb.github.io#dom-gpucomputepassencoder-dispatchworkgroupsindirect) method.Parameter Type Nullable Optional Description `indirectBuffer`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ Buffer containing the [indirect dispatch parameters](https://gpuweb.github.io#indirect-dispatch-parameters). `indirectOffset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✘ Offset in bytes into `indirectBuffer` where the dispatch data begins. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. Let `usageScope` be an empty [usage scope](https://gpuweb.github.io#usage-scope).
  3. For each `bindGroup` in `this`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)[merge](https://gpuweb.github.io#abstract-opdef-usage-scope-merge) `bindGroup`.[\[\[usedResources\]\]](https://gpuweb.github.io#dom-gpubindgroup-usedresources-slot)`this`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)
  4. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) `indirectBuffer` to `usageScope` with usage [input](https://gpuweb.github.io#internal-usage-input).
  5. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `usageScope` must satisfy [usage scope validation](https://gpuweb.github.io#usage-scope-validation).
    - [Validate encoder bind groups](https://gpuweb.github.io#abstract-opdef-validate-encoder-bind-groups)(`this`, `this`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)`true`.
    - [Validate immediate data](https://gpuweb.github.io#abstract-opdef-validate-immediate-data)(`this`, `this`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)`true`.
    - `indirectBuffer` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `indirectBuffer`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)
    - `indirectOffset` + sizeof([indirect dispatch parameters](https://gpuweb.github.io#indirect-dispatch-parameters)) ≤ `indirectBuffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)
    - `indirectOffset` is a multiple of 4.
  6. Let `bindingState` be a snapshot of `this`’s current state.
  7. [Enqueue a command](https://gpuweb.github.io#abstract-opdef-enqueue-a-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline).
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `workgroupCountX` be an unsigned 32-bit integer read from `indirectBuffer` at `indirectOffset` bytes.
  2. Let `workgroupCountY` be an unsigned 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 4) bytes.
  3. Let `workgroupCountZ` be an unsigned 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 8) bytes.
  4. Let `workgroupSize` be the computed workgroup size for `bindingState`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)
  5. If `workgroupCountX`, `workgroupCountY`, or `workgroupCountZ` is greater than `this`.device.limits.[maxComputeWorkgroupsPerDimension](https://gpuweb.github.io#dom-supported-limits-maxcomputeworkgroupsperdimension)
  6. If the entry point uses the [workgroup\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-workgroup_index) built-in value and `workgroupCountX` × `workgroupCountY` × `workgroupCountZ` is out of range of an unsigned
32-bit integer return.
  7. If the entry point uses the [global\_invocation\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-global_invocation_index) built-in value and `workgroupCountX` × `workgroupCountY` × `workgroupCountZ` × `workgroupSize` is out of range of an unsigned
32-bit integer return.
  8. Execute a grid of workgroups with dimensions \[`workgroupCountX`, `workgroupCountY`, `workgroupCountZ`\] with `bindingState`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-pipeline-slot)`bindingState`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)

#### 16.1.3. Finalization

The compute pass encoder can be ended by calling [end()](https://gpuweb.github.io#dom-gpucomputepassencoder-end)[end()](https://gpuweb.github.io#dom-gpucomputepassencoder-end)

- `end()`
- Completes recording of the compute pass commands sequence. **Called on:** [GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)
`this`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Let `parentEncoder` be `this`.[\[\[command\_encoder\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-command_encoder-slot)
  2. If any of the following requirements are unmet,
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error) and return.
  3. Set `this`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[ended](https://gpuweb.github.io#encoder-state-ended)".
  4. Set `parentEncoder`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[open](https://gpuweb.github.io#encoder-state-open)".
  5. [Extend](https://infra.spec.whatwg.org/#set-extend) `parentEncoder`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-used_bind_groups-slot)`this`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-used_bind_groups-slot)
  6. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `parentEncoder` and return.
    - `this` must be [valid](https://gpuweb.github.io#abstract-opdef-valid).
    - `this`.[\[\[debug\_group\_stack\]\]](https://gpuweb.github.io#dom-gpudebugcommandsmixin-debug_group_stack-slot)[be empty](https://infra.spec.whatwg.org/#list-is-empty).
  7. [Extend](https://infra.spec.whatwg.org/#list-extend) `parentEncoder`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)`this`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)
  8. If `this`.[\[\[endTimestampWrite\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-endtimestampwrite-slot)`null`:
    1. [Extend](https://infra.spec.whatwg.org/#list-extend) `parentEncoder`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)`this`.[\[\[endTimestampWrite\]\]](https://gpuweb.github.io#dom-gpucomputepassencoder-endtimestampwrite-slot)

## 17. Render Passes

### 17.1. `GPURenderPassEncoder`

\[Exposed Window ,Worker ),SecureContext interface GPURenderPassEncoder undefined setViewport float x float y float width float height float minDepth float maxDepth undefined setScissorRect GPUIntegerCoordinate x GPUIntegerCoordinate y GPUIntegerCoordinate width GPUIntegerCoordinate height undefined setBlendConstant GPUColor color undefined setStencilReference GPUStencilValue reference undefined beginOcclusionQuery GPUSize32 queryIndex undefined endOcclusionQuery undefined executeBundles sequence GPURenderBundle bundles undefined end GPURenderPassEncoder includes GPUObjectBase GPURenderPassEncoder includes GPUCommandsMixin GPURenderPassEncoder includes GPUDebugCommandsMixin GPURenderPassEncoder includes GPUBindingCommandsMixin GPURenderPassEncoder includes GPURenderCommandsMixin 

[GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[command_encoder]]`, of type [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
- The [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)
- `[[attachment_size]]`, readonly
- Set to the following extents: 
  - `width, height` = the dimensions of the pass’s render attachments
- `[[occlusion_query_set]]`, of type [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
- The [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)[GPURenderPassDescriptor](https://gpuweb.github.io#dictdef-gpurenderpassdescriptor)[occlusionQuerySet](https://gpuweb.github.io#dom-gpurenderpassdescriptor-occlusionqueryset)
- `[[endTimestampWrite]]`, of type [GPU command](https://gpuweb.github.io#gpu-command)?, readonly, defaulting to `null`
- [GPU command](https://gpuweb.github.io#gpu-command), if any, writing a timestamp when the pass ends.
- `[[maxDrawCount]]` of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The maximum number of draws allowed in this pass.
- `[[occlusion_query_active]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- Whether the pass’s [\[\[occlusion\_query\_set\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_set-slot)

When executing encoded render pass commands as part of a [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)

[RenderState](https://gpuweb.github.io#renderstate) has the following [queue timeline properties](https://gpuweb.github.io#queue-timeline-property):

- `[[occlusionQueryIndex]]`, of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- The index into [\[\[occlusion\_query\_set\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_set-slot)
- `[[viewport]]`
- Current viewport rectangle and depth range. Initially set to the following values: 
  - `x, y` = `0.0, 0.0`
  - `width, height` = the dimensions of the pass’s render targets
  - `minDepth, maxDepth` = `0.0, 1.0`
- `[[scissorRect]]`
- Current scissor rectangle. Initially set to the following values: 
  - `x, y` = `0, 0`
  - `width, height` = the dimensions of the pass’s render targets
- `[[blendConstant]]`, of type [GPUColor](https://gpuweb.github.io#typedefdef-gpucolor)
- Current blend constant value, initially `[0, 0, 0, 0]`.
- `[[stencilReference]]`, of type [GPUStencilValue](https://gpuweb.github.io#typedefdef-gpustencilvalue)
- Current stencil reference value, initially `0`.
- `[[colorAttachments]]`, of type [sequence](https://webidl.spec.whatwg.org/#idl-sequence)\<[GPURenderPassColorAttachment](https://gpuweb.github.io#dictdef-gpurenderpasscolorattachment)
- The color attachments and state for this render pass.
- `[[depthStencilAttachment]]`, of type [GPURenderPassDepthStencilAttachment](https://gpuweb.github.io#dictdef-gpurenderpassdepthstencilattachment)
- The depth/stencil attachment and state for this render pass.

Render passes also have framebuffer memory, which contains the [texel](https://gpuweb.github.io#texel-block) data associated with
each attachment that is written into by draw commands and read from for blending and depth/stencil testing.

Note: Depending on the GPU hardware, [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) may be the memory allocated by the attachment textures or
may be a separate area of memory that the texture data is copied to and from, such as with tile-based architectures.

#### 17.1.1. Render Pass Encoder Creation

dictionary GPURenderPassTimestampWrites required GPUQuerySet querySet GPUSize32 beginningOfPassWriteIndex GPUSize32 endOfPassWriteIndex 

- `querySet`,  of type [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
- The [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)["timestamp"](https://gpuweb.github.io#dom-gpuquerytype-timestamp)
- `beginningOfPassWriteIndex`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- If defined, indicates the query index in [querySet](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-queryset)
- `endOfPassWriteIndex`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- If defined, indicates the query index in [querySet](https://gpuweb.github.io#dom-gpurenderpasstimestampwrites-queryset)

Note: Timestamp query values are written in nanoseconds, but how the value is determined is
[implementation-defined](https://infra.spec.whatwg.org/#implementation-defined). See [§ 20.4 Timestamp Query](https://gpuweb.github.io#timestamp) for details.

dictionary GPURenderPassDescriptor GPUObjectDescriptorBase required sequence GPURenderPassColorAttachment colorAttachments GPURenderPassDepthStencilAttachment depthStencilAttachment GPUQuerySet occlusionQuerySet GPURenderPassTimestampWrites timestampWrites GPUSize64 maxDrawCount 

- `colorAttachments`,  of type `sequence<GPURenderPassColorAttachment?>`
- The set of [GPURenderPassColorAttachment](https://gpuweb.github.io#dictdef-gpurenderpasscolorattachment)
Due to [usage compatibility](https://gpuweb.github.io#compatible-usage-list), no color attachment
may alias another attachment or any resource used inside the render pass.
- `depthStencilAttachment`,  of type [GPURenderPassDepthStencilAttachment](https://gpuweb.github.io#dictdef-gpurenderpassdepthstencilattachment)
- The [GPURenderPassDepthStencilAttachment](https://gpuweb.github.io#dictdef-gpurenderpassdepthstencilattachment)
Due to [usage compatibility](https://gpuweb.github.io#compatible-usage-list), no writable depth/stencil attachment
may alias another attachment or any resource used inside the render pass.
- `occlusionQuerySet`,  of type [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
- The [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
- `timestampWrites`,  of type [GPURenderPassTimestampWrites](https://gpuweb.github.io#dictdef-gpurenderpasstimestampwrites)
- Defines which timestamp values will be written for this pass, and where to write them to.
- `maxDrawCount`,  of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64), defaulting to `50000000`
- The maximum number of draw calls that will be done in the render pass. Used by some implementations to size work injected before the render pass. Keeping the default value is a good default, unless it is known that more draw calls will be done.

Given a [GPUDevice](https://gpuweb.github.io#gpudevice)`device` and [GPURenderPassDescriptor](https://gpuweb.github.io#dictdef-gpurenderpassdescriptor)`this`, the following validation rules apply:

1. `this`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)[size](https://infra.spec.whatwg.org/#list-size) must be ≤ `device`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxColorAttachments](https://gpuweb.github.io#dom-supported-limits-maxcolorattachments)
2. For each non-`null` `colorAttachment` in `this`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
  1. `colorAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `device`.
  2. If `colorAttachment`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)[provided](https://infra.spec.whatwg.org/#map-exists):
    1. `colorAttachment`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)[valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `device`.
  3. `colorAttachment` must meet the [GPURenderPassColorAttachment Valid Usage](https://gpuweb.github.io#abstract-opdef-gpurenderpasscolorattachment-gpurenderpasscolorattachment-valid-usage) rules.
3. If `this`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)[provided](https://infra.spec.whatwg.org/#map-exists):
  1. `this`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)[valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `device`.
  2. `this`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)[GPURenderPassDepthStencilAttachment Valid Usage](https://gpuweb.github.io#abstract-opdef-gpurenderpassdepthstencilattachment-gpurenderpassdepthstencilattachment-valid-usage) rules.
4. There must exist at least one attachment, either: 
  - A non-`null` value in `this`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
  - A `this`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
5. [Validating GPURenderPassDescriptor’s color attachment bytes per sample](https://gpuweb.github.io#abstract-opdef-validating-gpurenderpassdescriptors-color-attachment-bytes-per-sample)(`device`, `this`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
6. All [view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)`null` members of `this`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)`this`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)
7. For each [view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)`null` members of `this`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)`this`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)[\[\[renderExtent\]\]](https://gpuweb.github.io#dom-gputextureview-renderextent-slot)
8. If `this`.[occlusionQuerySet](https://gpuweb.github.io#dom-gpurenderpassdescriptor-occlusionqueryset)[provided](https://infra.spec.whatwg.org/#map-exists):
  1. `this`.[occlusionQuerySet](https://gpuweb.github.io#dom-gpurenderpassdescriptor-occlusionqueryset)[valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `device`.
  2. `this`.[occlusionQuerySet](https://gpuweb.github.io#dom-gpurenderpassdescriptor-occlusionqueryset)[type](https://gpuweb.github.io#dom-gpuqueryset-type)[occlusion](https://gpuweb.github.io#dom-gpuquerytype-occlusion)
9. If `this`.[timestampWrites](https://gpuweb.github.io#dom-gpurenderpassdescriptor-timestampwrites)[provided](https://infra.spec.whatwg.org/#map-exists):
  - [Validate timestampWrites](https://gpuweb.github.io#abstract-opdef-validate-timestampwrites)(`device`, `this`.[timestampWrites](https://gpuweb.github.io#dom-gpurenderpassdescriptor-timestampwrites)

`device`,

`colorAttachments`)

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`
- [sequence](https://webidl.spec.whatwg.org/#idl-sequence)\<[GPURenderPassColorAttachment](https://gpuweb.github.io#dictdef-gpurenderpasscolorattachment)`colorAttachments`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `formats` be an empty [list](https://infra.spec.whatwg.org/#list)\<[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
2. For each `colorAttachment` in `colorAttachments`:
  1. If `colorAttachment` is `undefined`, continue.
  2. [Append](https://infra.spec.whatwg.org/#list-append) `colorAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)`formats`.
3. [Calculating color attachment bytes per sample](https://gpuweb.github.io#abstract-opdef-calculating-color-attachment-bytes-per-sample)(`formats`) must be ≤ `device`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxColorAttachmentBytesPerSample](https://gpuweb.github.io#dom-supported-limits-maxcolorattachmentbytespersample)

##### 17.1.1.1. Color Attachments

dictionary GPURenderPassColorAttachment required (GPUTexture or GPUTextureView view GPUIntegerCoordinate depthSlice GPUTexture or GPUTextureView resolveTarget GPUColor clearValue required GPULoadOp loadOp required GPUStoreOp storeOp 

- `view`,  of type `(GPUTexture or GPUTextureView)`
- Describes the texture [subresource](https://gpuweb.github.io#subresource) that will be output to for this color attachment.
The [subresource](https://gpuweb.github.io#subresource) is determined by calling [get as texture view](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)([view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)
- `depthSlice`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate)
- Indicates the depth slice index of ["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)
- `resolveTarget`,  of type `(GPUTexture or GPUTextureView)`
- Describes the texture [subresource](https://gpuweb.github.io#subresource) that will receive the resolved output for this color
attachment if [view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[subresource](https://gpuweb.github.io#subresource) is determined by calling [get as texture view](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)([resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)
- `clearValue`,  of type [GPUColor](https://gpuweb.github.io#typedefdef-gpucolor)
- Indicates the value to clear [view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[provided](https://infra.spec.whatwg.org/#map-exists), defaults to `{r: 0, g: 0, b: 0, a: 0}`. Ignored
if [loadOp](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-loadop)["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
The components of [clearValue](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-clearvalue)[to a texel value of texture format](https://gpuweb.github.io#abstract-opdef-to-a-texel-value-of-texture-format) matching the render attachment.
If conversion fails, a validation error is generated.
- `loadOp`,  of type [GPULoadOp](https://gpuweb.github.io#enumdef-gpuloadop)
- Indicates the load operation to perform on [view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)
Note: It is recommended to prefer clearing; see ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
- `storeOp`,  of type [GPUStoreOp](https://gpuweb.github.io#enumdef-gpustoreop)
- The store operation to perform on [view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)

Given a [GPURenderPassColorAttachment](https://gpuweb.github.io#dictdef-gpurenderpasscolorattachment)`this`:

1. Let `renderViewDescriptor` be `this`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)
2. Let `renderTexture` be `this`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)
3. All of the requirements in the following steps `must` be met.
  1. `renderViewDescriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)`must` be a [color renderable format](https://gpuweb.github.io#color-renderable-format).
  2. `this`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)`must` be a [renderable texture view](https://gpuweb.github.io#abstract-opdef-renderable-texture-view).
  3. If `renderViewDescriptor`.[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
    1. `this`.[depthSlice](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-depthslice)`must` [be provided](https://infra.spec.whatwg.org/#map-exists) and `must` be \< the [depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) of the [logical miplevel-specific texture extent](https://gpuweb.github.io#logical-miplevel-specific-texture-extent) of the `renderTexture` [subresource](https://gpuweb.github.io#subresource) at [mipmap level](https://gpuweb.github.io#mipmap-level) `renderViewDescriptor`.[baseMipLevel](https://gpuweb.github.io#dom-gputextureviewdescriptor-basemiplevel)
Otherwise: 
    1. `this`.[depthSlice](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-depthslice)`must` not [be provided](https://infra.spec.whatwg.org/#map-exists).
  4. If `renderViewDescriptor`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)[TRANSIENT\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-transient_attachment)
  5. If `this`.[loadOp](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-loadop)["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
    1. Converting the IDL value `this`.[clearValue](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-clearvalue)[to a texel value of texture format](https://gpuweb.github.io#abstract-opdef-to-a-texel-value-of-texture-format) `renderViewDescriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)`must` not throw a [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)
Note: An error is not thrown if the value is out-of-range for the format but in-range for
the corresponding WGSL primitive type (`f32`, `i32`, or `u32`).
  6. If `this`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)[provided](https://infra.spec.whatwg.org/#map-exists):
    1. Let `resolveViewDescriptor` be `this`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)
    2. Let `resolveTexture` be `this`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)
    3. `renderTexture`.[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)`must` be > 1.
    4. `resolveTexture`.[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)`must` be 1.
    5. `this`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)`must` be a non-3d [renderable texture view](https://gpuweb.github.io#abstract-opdef-renderable-texture-view).
    6. `this`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)[\[\[renderExtent\]\]](https://gpuweb.github.io#dom-gputextureview-renderextent-slot)`this`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[\[\[renderExtent\]\]](https://gpuweb.github.io#dom-gputextureview-renderextent-slot)`must` match.
    7. `resolveViewDescriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)`must` equal `renderViewDescriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)
    8. `resolveTexture`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)`must` equal `renderTexture`.[format](https://gpuweb.github.io#dom-gputexturedescriptor-format)
    9. `resolveViewDescriptor`.[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)`must` support resolve according to [§ 26.1.1 Plain color formats](https://gpuweb.github.io#plain-color-formats).
    10. `resolveViewDescriptor`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)`must` not include the [TRANSIENT\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-transient_attachment)

[GPUTextureView](https://gpuweb.github.io#gputextureview)
`view`is a renderable texture view if the all of the requirements in the following

[device timeline](https://gpuweb.github.io#device-timeline)steps are met:

1. Let `descriptor` be `view`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)
2. `descriptor`.[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)
3. `descriptor`.[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)["2d"](https://gpuweb.github.io#dom-gputextureviewdimension-2d)["2d-array"](https://gpuweb.github.io#dom-gputextureviewdimension-2d-array)["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
4. `descriptor`.[mipLevelCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-miplevelcount)
5. `descriptor`.[arrayLayerCount](https://gpuweb.github.io#dom-gputextureviewdescriptor-arraylayercount)
6. `descriptor`.[aspect](https://gpuweb.github.io#dom-gputextureviewdescriptor-aspect)[aspects](https://gpuweb.github.io#aspect) of `view`.[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)
7. `descriptor`.[swizzle](https://gpuweb.github.io#dom-gputextureviewdescriptor-swizzle)`"rgba"`.

`formats`)

**Arguments:**

- [sequence](https://webidl.spec.whatwg.org/#idl-sequence)\<[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)`formats`

**Returns:** [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)

1. Let `total` be 0.
2. For each non-null `format` in `formats`
  1. [Assert](https://infra.spec.whatwg.org/#assert): `format` is a [color renderable format](https://gpuweb.github.io#color-renderable-format).
  2. Let `renderTargetPixelByteCost` be the [render target pixel byte cost](https://gpuweb.github.io#render-target-pixel-byte-cost) of `format`.
  3. Let `renderTargetComponentAlignment` be the [render target component alignment](https://gpuweb.github.io#render-target-component-alignment) of `format`.
  4. Round `total` up to the smallest multiple of `renderTargetComponentAlignment` greater than or equal to `total`.
  5. Add `renderTargetPixelByteCost` to `total`.
3. Return `total`.

##### 17.1.1.2. Depth/Stencil Attachments

dictionary GPURenderPassDepthStencilAttachment required (GPUTexture or GPUTextureView view float depthClearValue GPULoadOp depthLoadOp GPUStoreOp depthStoreOp boolean depthReadOnly false ;GPUStencilValue stencilClearValue GPULoadOp stencilLoadOp GPUStoreOp stencilStoreOp boolean stencilReadOnly false ; };

- `view`,  of type `(GPUTexture or GPUTextureView)`
- Describes the texture [subresource](https://gpuweb.github.io#subresource) that will be output to and read from for this
depth/stencil attachment.
The [subresource](https://gpuweb.github.io#subresource) is determined by calling [get as texture view](https://gpuweb.github.io#abstract-opdef-get-as-texture-view)([view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
- `depthClearValue`,  of type [float](https://webidl.spec.whatwg.org/#idl-float)
- Indicates the value to clear [view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)[depthLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthloadop)["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
- `depthLoadOp`,  of type [GPULoadOp](https://gpuweb.github.io#enumdef-gpuloadop)
- Indicates the load operation to perform on [view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
Note: It is recommended to prefer clearing; see ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
- `depthStoreOp`,  of type [GPUStoreOp](https://gpuweb.github.io#enumdef-gpustoreop)
- The store operation to perform on [view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
- `depthReadOnly`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- Indicates that the depth component of [view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
- `stencilClearValue`,  of type [GPUStencilValue](https://gpuweb.github.io#typedefdef-gpustencilvalue), defaulting to `0`
- Indicates the value to clear [view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)[stencilLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilloadop)["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
The value will be converted to the type of the stencil aspect of `view` by taking the same
number of LSBs as the number of bits in the stencil aspect of one [texel](https://gpuweb.github.io#texel-block) of `view`.
- `stencilLoadOp`,  of type [GPULoadOp](https://gpuweb.github.io#enumdef-gpuloadop)
- Indicates the load operation to perform on [view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
Note: It is recommended to prefer clearing; see ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
- `stencilStoreOp`,  of type [GPUStoreOp](https://gpuweb.github.io#enumdef-gpustoreop)
- The store operation to perform on [view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
- `stencilReadOnly`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- Indicates that the stencil component of [view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)

Given a [GPURenderPassDepthStencilAttachment](https://gpuweb.github.io#dictdef-gpurenderpassdepthstencilattachment)`this`:

1. Let `format` be `this`.[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)
2. Let `usage` be `this`.[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[usage](https://gpuweb.github.io#dom-gputextureviewdescriptor-usage)
3. All of the requirements in the following steps `must` be met.
  1. `this`.[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)`must` have a [depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format).
  2. `this`.[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)`must` be a [renderable texture view](https://gpuweb.github.io#abstract-opdef-renderable-texture-view).
  3. If `this`.[depthLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthloadop)["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)`this`.[depthClearValue](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthclearvalue)`must` [be provided](https://infra.spec.whatwg.org/#map-exists) and `must` be between 0.0 and 1.0,
inclusive.
  4. If `format` has a [depth](https://gpuweb.github.io#aspect-depth) aspect and `this`.[depthReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthreadonly)`false`:
    1. `this`.[depthLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthloadop)`must` [be provided](https://infra.spec.whatwg.org/#map-exists).
    2. `this`.[depthStoreOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthstoreop)`must` [be provided](https://infra.spec.whatwg.org/#map-exists).
Otherwise: 
    1. `this`.[depthLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthloadop)`must` not [be provided](https://infra.spec.whatwg.org/#map-exists).
    2. `this`.[depthStoreOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthstoreop)`must` not [be provided](https://infra.spec.whatwg.org/#map-exists).
  5. If `format` has a [stencil](https://gpuweb.github.io#aspect-stencil) aspect and `this`.[stencilReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilreadonly)`false`:
    1. `this`.[stencilLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilloadop)`must` [be provided](https://infra.spec.whatwg.org/#map-exists).
    2. `this`.[stencilStoreOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilstoreop)`must` [be provided](https://infra.spec.whatwg.org/#map-exists).
Otherwise: 
    1. `this`.[stencilLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilloadop)`must` not [be provided](https://infra.spec.whatwg.org/#map-exists).
    2. `this`.[stencilStoreOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilstoreop)`must` not [be provided](https://infra.spec.whatwg.org/#map-exists).
  6. If `usage` includes the [TRANSIENT\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-transient_attachment)
    1. If `format` has a [depth](https://gpuweb.github.io#aspect-depth) aspect:
      1. `this`.[depthLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthloadop)`must` be ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
      2. `this`.[depthStoreOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthstoreop)`must` be ["discard"](https://gpuweb.github.io#dom-gpustoreop-discard)
    2. If `format` has a [stencil](https://gpuweb.github.io#aspect-stencil) aspect:
      1. `this`.[stencilLoadOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilloadop)`must` be ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)
      2. `this`.[stencilStoreOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilstoreop)`must` be ["discard"](https://gpuweb.github.io#dom-gpustoreop-discard)

##### 17.1.1.3. Load & Store Operations

enum GPULoadOp "load" "clear" 

- `"load"`
- Loads the existing value for this attachment into the render pass.
- `"clear"`
- Loads a clear value for this attachment into the render pass. Note:
On some GPU hardware (primarily mobile), ["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)["clear"](https://gpuweb.github.io#dom-gpuloadop-clear)["load"](https://gpuweb.github.io#dom-gpuloadop-load)

enum GPUStoreOp "store" "discard" 

- `"store"`
- Stores the resulting value of the render pass for this attachment.
- `"discard"`
- Discards the resulting value of the render pass for this attachment. Note: Discarded attachments
behave as if they are cleared to zero, but implementations are not required to perform a
clear at the end of the render pass. Implementations which do not explicitly clear discarded
attachments at the end of a pass must lazily clear them prior to the reading the attachment
contents, which occurs via sampling, copies, attaching to a later render pass with
["load"](https://gpuweb.github.io#dom-gpuloadop-load)[get a copy of the image contents of a context](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context)), etc.

##### 17.1.1.4. Render Pass Layout

[GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)[GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)[GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[layouts](https://gpuweb.github.io#abstract-opdef-derive-render-targets-layout-from-pass) and
[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)[layouts](https://gpuweb.github.io#abstract-opdef-derive-render-targets-layout-from-pipeline).
It determines compatibility between render passes, render bundles, and render pipelines.

dictionary GPURenderPassLayout GPUObjectDescriptorBase required sequence GPUTextureFormat colorFormats GPUTextureFormat depthStencilFormat GPUSize32 sampleCount 

- `colorFormats`,  of type `sequence<GPUTextureFormat?>`
- A list of the [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- `depthStencilFormat`,  of type [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- The [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- `sampleCount`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32), defaulting to `1`
- Number of samples per pixel in the attachments for this pass or bundle.

[GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)
- Their [depthStencilFormat](https://gpuweb.github.io#dom-gpurenderpasslayout-depthstencilformat)[sampleCount](https://gpuweb.github.io#dom-gpurenderpasslayout-samplecount)
- Their [colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)`null`s.

**Arguments:**

- [GPURenderPassDescriptor](https://gpuweb.github.io#dictdef-gpurenderpassdescriptor)`descriptor`

**Returns:** [GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `layout` be a new [GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)
2. For each `colorAttachment` in `descriptor`.[colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
  1. If `colorAttachment` is not `null`:
    1. Set `layout`.[sampleCount](https://gpuweb.github.io#dom-gpurenderpasslayout-samplecount)`colorAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)
    2. Append `colorAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)`layout`.[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)
Otherwise: 
    1. Append `null` to `layout`.[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)
3. Let `depthStencilAttachment` be `descriptor`.[depthStencilAttachment](https://gpuweb.github.io#dom-gpurenderpassdescriptor-depthstencilattachment)
4. If `depthStencilAttachment` is not `null`:
  1. Let `view` be `depthStencilAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-view)
  2. Set `layout`.[sampleCount](https://gpuweb.github.io#dom-gpurenderpasslayout-samplecount)`view`.[\[\[texture\]\]](https://gpuweb.github.io#dom-gputextureview-texture-slot)[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)
  3. Set `layout`.[depthStencilFormat](https://gpuweb.github.io#dom-gpurenderpasslayout-depthstencilformat)`view`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[format](https://gpuweb.github.io#dom-gputextureviewdescriptor-format)
5. Return `layout`.

**Arguments:**

- [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)`descriptor`

**Returns:** [GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `layout` be a new [GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)
2. Set `layout`.[sampleCount](https://gpuweb.github.io#dom-gpurenderpasslayout-samplecount)`descriptor`.[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)[count](https://gpuweb.github.io#dom-gpumultisamplestate-count)
3. If `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)[provided](https://infra.spec.whatwg.org/#map-exists):
  1. Set `layout`.[depthStencilFormat](https://gpuweb.github.io#dom-gpurenderpasslayout-depthstencilformat)`descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)[format](https://gpuweb.github.io#dom-gpudepthstencilstate-format)
4. If `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)[provided](https://infra.spec.whatwg.org/#map-exists):
  1. For each `colorTarget` in `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)
    1. Append `colorTarget`.[format](https://gpuweb.github.io#dom-gpucolortargetstate-format)`layout`.[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)`colorTarget` is not `null`, or append `null` otherwise.
5. Return `layout`.

#### 17.1.2. Finalization

The render pass encoder can be ended by calling [end()](https://gpuweb.github.io#dom-gpurenderpassencoder-end)[end()](https://gpuweb.github.io#dom-gpurenderpassencoder-end)

- `end()`
- Completes recording of the render pass commands sequence. **Called on:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
`this`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Let `parentEncoder` be `this`.[\[\[command\_encoder\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-command_encoder-slot)
  2. If any of the following requirements are unmet,
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error) and return.
  3. Set `this`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[ended](https://gpuweb.github.io#encoder-state-ended)".
  4. Set `parentEncoder`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[open](https://gpuweb.github.io#encoder-state-open)".
  5. [Extend](https://infra.spec.whatwg.org/#set-extend) `parentEncoder`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-used_bind_groups-slot)`this`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-used_bind_groups-slot)
  6. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `parentEncoder` and return.
    - `this` must be [valid](https://gpuweb.github.io#abstract-opdef-valid).
    - `this`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[usage scope validation](https://gpuweb.github.io#usage-scope-validation).
    - `this`.[\[\[debug\_group\_stack\]\]](https://gpuweb.github.io#dom-gpudebugcommandsmixin-debug_group_stack-slot)[be empty](https://infra.spec.whatwg.org/#list-is-empty).
    - `this`.[\[\[occlusion\_query\_active\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_active-slot)`false`.
    - `this`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)`this`.[\[\[maxDrawCount\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-maxdrawcount-slot)
  7. [Extend](https://infra.spec.whatwg.org/#list-extend) `parentEncoder`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)`this`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)
  8. If `this`.[\[\[endTimestampWrite\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-endtimestampwrite-slot)`null`:
    1. [Extend](https://infra.spec.whatwg.org/#list-extend) `parentEncoder`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)`this`.[\[\[endTimestampWrite\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-endtimestampwrite-slot)
  9. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. For each non-`null` `colorAttachment` in `renderState`.[\[\[colorAttachments\]\]](https://gpuweb.github.io#dom-renderstate-colorattachments-slot)
    1. Let `colorView` be `colorAttachment`.[view](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-view)
    2. If `colorView`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gputextureview-descriptor-slot)[dimension](https://gpuweb.github.io#dom-gputextureviewdescriptor-dimension)
      - ["3d"](https://gpuweb.github.io#dom-gputextureviewdimension-3d)
      - Let `colorSubregion` be `colorAttachment`.[depthSlice](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-depthslice)`colorView`.
      - Otherwise
      - Let `colorSubregion` be `colorView`.
    3. If `colorAttachment`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)`null`:
      1. Resolve the multiple samples of every [texel](https://gpuweb.github.io#texel-block) of `colorSubregion` to a single
sample and copy to `colorAttachment`.[resolveTarget](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-resolvetarget)
    4. If `colorAttachment`.[storeOp](https://gpuweb.github.io#dom-gpurenderpasscolorattachment-storeop)
      - ["store"](https://gpuweb.github.io#dom-gpustoreop-store)
      - Ensure the contents of the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with `colorSubregion` are stored in `colorSubregion`.
      - ["discard"](https://gpuweb.github.io#dom-gpustoreop-discard)
      - Set every [texel](https://gpuweb.github.io#texel-block) of `colorSubregion` to zero.
  2. Let `depthStencilAttachment` be `renderState`.[\[\[depthStencilAttachment\]\]](https://gpuweb.github.io#dom-renderstate-depthstencilattachment-slot)
  3. If `depthStencilAttachment` is not `null`:
    1. If `depthStencilAttachment`.[depthStoreOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthstoreop)
      - Not [provided](https://infra.spec.whatwg.org/#map-exists)
      - [Assert](https://infra.spec.whatwg.org/#assert) that `depthStencilAttachment`.[depthReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-depthreadonly)`true` and leave the [depth](https://gpuweb.github.io#aspect-depth) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` unchanged.
      - ["store"](https://gpuweb.github.io#dom-gpustoreop-store)
      - Ensure the contents of the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with the [depth](https://gpuweb.github.io#aspect-depth) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` are stored in `depthStencilView`.
      - ["discard"](https://gpuweb.github.io#dom-gpustoreop-discard)
      - Set every [texel](https://gpuweb.github.io#texel-block) in the [depth](https://gpuweb.github.io#aspect-depth) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` to zero.
    2. If `depthStencilAttachment`.[stencilStoreOp](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilstoreop)
      - Not [provided](https://infra.spec.whatwg.org/#map-exists)
      - [Assert](https://infra.spec.whatwg.org/#assert) that `depthStencilAttachment`.[stencilReadOnly](https://gpuweb.github.io#dom-gpurenderpassdepthstencilattachment-stencilreadonly)`true` and leave the [stencil](https://gpuweb.github.io#aspect-stencil) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` unchanged.
      - ["store"](https://gpuweb.github.io#dom-gpustoreop-store)
      - Ensure the contents of the [framebuffer memory](https://gpuweb.github.io#framebuffer-memory) associated with the [stencil](https://gpuweb.github.io#aspect-stencil) [subresource](https://gpuweb.github.io#gputextureview-subresources) of `depthStencilView` are stored in `depthStencilView`.
      - ["discard"](https://gpuweb.github.io#dom-gpustoreop-discard)
      - Set every [texel](https://gpuweb.github.io#texel-block) in the [stencil](https://gpuweb.github.io#aspect-stencil) [subresource](https://gpuweb.github.io#gputextureview-subresources) `depthStencilView` to zero.
  4. Let `renderState` be `null`.
Note: Discarded attachments behave as if they are cleared to zero, but implementations are not required
        to perform a clear at the end of the render pass. See the note on ["discard"](https://gpuweb.github.io#dom-gpustoreop-discard)
Note: [Read-only depth-stencil](https://gpuweb.github.io#read-only-depth-stencil) attachments can be thought of as implicitly using the ["store"](https://gpuweb.github.io#dom-gpustoreop-store)[GPURenderPassDepthStencilAttachment Valid Usage](https://gpuweb.github.io#abstract-opdef-gpurenderpassdepthstencilattachment-gpurenderpassdepthstencilattachment-valid-usage).

### 17.2. `GPURenderCommandsMixin`

[GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)[GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)

interface mixin GPURenderCommandsMixin undefined setPipeline GPURenderPipeline pipeline undefined setIndexBuffer GPUBuffer buffer GPUIndexFormat indexFormat optional GPUSize64 offset optional GPUSize64 size undefined setVertexBuffer GPUIndex32 slot GPUBuffer buffer optional GPUSize64 offset optional GPUSize64 size undefined draw GPUSize32 vertexCount optional GPUSize32 instanceCount optional GPUSize32 firstVertex optional GPUSize32 firstInstance undefined drawIndexed GPUSize32 indexCount optional GPUSize32 instanceCount optional GPUSize32 firstIndex optional GPUSignedOffset32 baseVertex optional GPUSize32 firstInstance undefined drawIndirect GPUBuffer indirectBuffer GPUSize64 indirectOffset undefined drawIndexedIndirect GPUBuffer indirectBuffer GPUSize64 indirectOffset 

[GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)[GPUObjectBase](https://gpuweb.github.io#gpuobjectbase)[GPUCommandsMixin](https://gpuweb.github.io#gpucommandsmixin)[GPUBindingCommandsMixin](https://gpuweb.github.io#gpubindingcommandsmixin)

[GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[layout]]`, of type [GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)
- The layout of the render pass.
- `[[depthReadOnly]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- If `true`, indicates that the depth component is not modified.
- `[[stencilReadOnly]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- If `true`, indicates that the stencil component is not modified.
- `[[usage scope]]`, of type [usage scope](https://gpuweb.github.io#usage-scope), initially empty
- The [usage scope](https://gpuweb.github.io#usage-scope) for this render pass or bundle.
- `[[pipeline]]`, of type [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
`null`
- The current [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
- `[[index_buffer]]`, of type [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`null`
- The current buffer to read index data from.
- `[[index_format]]`, of type [GPUIndexFormat](https://gpuweb.github.io#enumdef-gpuindexformat)
- The format of the index data in [\[\[index\_buffer\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer-slot)
- `[[index_buffer_offset]]`, of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The offset in bytes of the section of [\[\[index\_buffer\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer-slot)
- `[[index_buffer_size]]`, of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The size in bytes of the section of [\[\[index\_buffer\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer-slot)`0`.
- `[[vertex_buffers]]`, of type [ordered map](https://infra.spec.whatwg.org/#ordered-map)\<slot, [GPUBuffer](https://gpuweb.github.io#gpubuffer)
- The current [GPUBuffer](https://gpuweb.github.io#gpubuffer)
- `[[vertex_buffer_sizes]]`, of type [ordered map](https://infra.spec.whatwg.org/#ordered-map)\<slot, [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The size in bytes of the section of [GPUBuffer](https://gpuweb.github.io#gpubuffer)
- `[[drawCount]]`, of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The number of draw commands recorded in this encoder.

[GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
`encoder`which issues the steps of a

[GPU Command](https://gpuweb.github.io#gpu-command)

`command`with

[RenderState](https://gpuweb.github.io#renderstate)

`renderState`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. [Append](https://infra.spec.whatwg.org/#list-append) `command` to `encoder`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)
2. When `command` is executed as part of a [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)`commandBuffer`:
  1. Issue the steps of `command` with `commandBuffer`.[\[\[renderState\]\]](https://gpuweb.github.io#dom-gpucommandbuffer-renderstate-slot)`renderState`.

#### 17.2.1. Drawing

- `setPipeline(pipeline)`
- Sets the current [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
**Called on:** [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
**Arguments:**
Arguments for the [GPURenderCommandsMixin.setPipeline(pipeline)](https://gpuweb.github.io#dom-gpurendercommandsmixin-setpipeline) method.Parameter Type Nullable Optional Description `pipeline`
[GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
✘ ✘ The render pipeline to use for subsequent drawing commands. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. Let `pipelineTargetsLayout` be [derive render targets layout from pipeline](https://gpuweb.github.io#abstract-opdef-derive-render-targets-layout-from-pipeline)(`pipeline`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)
  3. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `pipeline` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `this`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-layout-slot)[equals](https://gpuweb.github.io#render-pass-layout-equals) `pipelineTargetsLayout`.
    - If `pipeline`.[\[\[writesDepth\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesdepth-slot)`this`.[\[\[depthReadOnly\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-depthreadonly-slot)`false`.
    - If `pipeline`.[\[\[writesStencil\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesstencil-slot)`this`.[\[\[stencilReadOnly\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-stencilreadonly-slot)`false`.
  4. Set `this`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)`pipeline`.
- `setIndexBuffer(buffer, indexFormat, offset, size)`
- Sets the current index buffer. **Called on:** [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
**Arguments:**
Arguments for the [GPURenderCommandsMixin.setIndexBuffer(buffer, indexFormat, offset, size)](https://gpuweb.github.io#dom-gpurendercommandsmixin-setindexbuffer) method.Parameter Type Nullable Optional Description `buffer`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ Buffer containing index data to use for subsequent drawing commands. `indexFormat`
[GPUIndexFormat](https://gpuweb.github.io#enumdef-gpuindexformat)
✘ ✘ Format of the index data contained in `buffer`. `offset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Offset in bytes into `buffer` where the index data begins. Defaults to `0`. `size`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Size in bytes of the index data in `buffer`.
Defaults to the size of the buffer minus the offset. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If `size` is missing, set `size` to max(0, `buffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)`offset`).
  3. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `buffer` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `buffer`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[INDEX](https://gpuweb.github.io#dom-gpubufferusage-index)
    - `offset` is a multiple of `indexFormat`’s byte size.
    - `offset` + `size` ≤ `buffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)
  4. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) `buffer` to [\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[input](https://gpuweb.github.io#internal-usage-input).
  5. Set `this`.[\[\[index\_buffer\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer-slot)`buffer`.
  6. Set `this`.[\[\[index\_format\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_format-slot)`indexFormat`.
  7. Set `this`.[\[\[index\_buffer\_offset\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer_offset-slot)`offset`.
  8. Set `this`.[\[\[index\_buffer\_size\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer_size-slot)`size`.
- `setVertexBuffer(slot, buffer, offset, size)`
- Sets the current vertex buffer for the given slot. **Called on:** [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
**Arguments:**
Arguments for the [GPURenderCommandsMixin.setVertexBuffer(slot, buffer, offset, size)](https://gpuweb.github.io#dom-gpurendercommandsmixin-setvertexbuffer) method.Parameter Type Nullable Optional Description `slot`
[GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)
✘ ✘ The vertex buffer slot to set the vertex buffer for. `buffer`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)?
✔ ✘ Buffer containing vertex data to use for subsequent drawing commands. `offset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Offset in bytes into `buffer` where the vertex data begins. Defaults to `0`. `size`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Size in bytes of the vertex data in `buffer`.
Defaults to the size of the buffer minus the offset. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. Let `bufferSize` be 0 if `buffer` is `null`, or `buffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)
  3. If `size` is missing, set `size` to max(0, `bufferSize` - `offset`).
  4. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `slot` must be \< `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxVertexBuffers](https://gpuweb.github.io#dom-supported-limits-maxvertexbuffers)
    - `offset` must be a multiple of 4.
    - `offset` + `size` must be ≤ `bufferSize`.
  5. If `buffer` is `null`:
    1. [Remove](https://infra.spec.whatwg.org/#map-remove) `this`.[\[\[vertex\_buffers\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffers-slot)`slot`\].
    2. [Remove](https://infra.spec.whatwg.org/#map-remove) `this`.[\[\[vertex\_buffer\_sizes\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffer_sizes-slot)`slot`\].
Otherwise: 
    1. If any of the following requirements are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
      - `buffer` must be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
      - `buffer`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[VERTEX](https://gpuweb.github.io#dom-gpubufferusage-vertex)
    2. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) `buffer` to [\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[input](https://gpuweb.github.io#internal-usage-input).
    3. Set `this`.[\[\[vertex\_buffers\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffers-slot)`slot`\] to be `buffer`.
    4. Set `this`.[\[\[vertex\_buffer\_sizes\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffer_sizes-slot)`slot`\] to be `size`.
- `draw(vertexCount, instanceCount, firstVertex, firstInstance)`
- Draws primitives.
See [§ 23.2 Rendering](https://gpuweb.github.io#rendering-operations) for the detailed specification.
**Called on:** [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
**Arguments:**
Arguments for the [GPURenderCommandsMixin.draw(vertexCount, instanceCount, firstVertex, firstInstance)](https://gpuweb.github.io#dom-gpurendercommandsmixin-draw) method.Parameter Type Nullable Optional Description `vertexCount`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✘ The number of vertices to draw. `instanceCount`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✔ The number of instances to draw. `firstVertex`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✔ Offset into the vertex buffers, in vertices, to begin drawing from. `firstInstance`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✔ First instance to draw. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. All of the requirements in the following steps `must` be met.
If any are unmet, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    1. It `must` be [valid to draw](https://gpuweb.github.io#abstract-opdef-valid-to-draw) with `this`.
    2. Let `buffers` be `this`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)
    3. For each [GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)`slot` from `0` to `buffers`.[size](https://infra.spec.whatwg.org/#list-size) (non-inclusive):
      1. If `buffers`\[`slot`\] is `null`, [continue](https://infra.spec.whatwg.org/#iteration-continue).
      2. Let `bufferSize` be `this`.[\[\[vertex\_buffer\_sizes\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffer_sizes-slot)`slot`\].
      3. Let `stride` be `buffers`\[`slot`\].[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)
      4. Let `attributes` be `buffers`\[`slot`\].[attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes)
      5. Let `lastStride` be the maximum value of
(`attribute`.[offset](https://gpuweb.github.io#dom-gpuvertexattribute-offset)[byteSize](https://gpuweb.github.io#abstract-opdef-gpuvertexformat-bytesize)(`attribute`.[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)`attribute` in `attributes`, or 0 if `attributes` is [empty](https://infra.spec.whatwg.org/#list-empty).
      6. Let `strideCount` be computed based on `buffers`\[`slot`\].[stepMode](https://gpuweb.github.io#dom-gpuvertexbufferlayout-stepmode)
        - ["vertex"](https://gpuweb.github.io#dom-gpuvertexstepmode-vertex)
        - `firstVertex` + `vertexCount`
        - ["instance"](https://gpuweb.github.io#dom-gpuvertexstepmode-instance)
        - `firstInstance` + `instanceCount`
      7. If `strideCount` ≠ `0`:
        1. (`strideCount` − `1`) × `stride` + `lastStride must` be ≤ `bufferSize`.
  3. Increment `this`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)
  4. Let `bindingState` be a snapshot of `this`’s current state.
  5. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Draw `instanceCount` instances, starting with instance `firstInstance`, of
primitives consisting of `vertexCount` vertices, starting with vertex `firstVertex`,
with the states from `bindingState` and `renderState`.
- `drawIndexed(indexCount, instanceCount, firstIndex, baseVertex, firstInstance)`
- Draws indexed primitives.
See [§ 23.2 Rendering](https://gpuweb.github.io#rendering-operations) for the detailed specification.
**Called on:** [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
**Arguments:**
Arguments for the [GPURenderCommandsMixin.drawIndexed(indexCount, instanceCount, firstIndex, baseVertex, firstInstance)](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindexed) method.Parameter Type Nullable Optional Description `indexCount`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✘ The number of indices to draw. `instanceCount`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✔ The number of instances to draw. `firstIndex`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✔ Offset into the index buffer, in indices, begin drawing from. `baseVertex`
[GPUSignedOffset32](https://gpuweb.github.io#typedefdef-gpusignedoffset32)
✘ ✔ Added to each index value before indexing into the vertex buffers. `firstInstance`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✔ First instance to draw. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - It is [valid to draw indexed](https://gpuweb.github.io#abstract-opdef-valid-to-draw-indexed) with `this`.
    - `firstIndex` + `indexCount` ≤ `this`.[\[\[index\_buffer\_size\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer_size-slot)`this`.[\[\[index\_format\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_format-slot)
    - Let `buffers` be `this`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)
    - For each [GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)`slot` from `0` to `buffers`.[size](https://infra.spec.whatwg.org/#list-size) (non-inclusive):
      - If `buffers`\[`slot`\] is `null`, [continue](https://infra.spec.whatwg.org/#iteration-continue).
      - Let `bufferSize` be `this`.[\[\[vertex\_buffer\_sizes\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffer_sizes-slot)`slot`\].
      - Let `stride` be `buffers`\[`slot`\].[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)
      - Let `lastStride` be max(`attribute`.[offset](https://gpuweb.github.io#dom-gpuvertexattribute-offset)[byteSize](https://gpuweb.github.io#abstract-opdef-gpuvertexformat-bytesize)(`attribute`.[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)`attribute` in `buffers`\[`slot`\].[attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes)
      - Let `strideCount` be `firstInstance` + `instanceCount`.
      - If `buffers`\[`slot`\].[stepMode](https://gpuweb.github.io#dom-gpuvertexbufferlayout-stepmode)["instance"](https://gpuweb.github.io#dom-gpuvertexstepmode-instance)`strideCount` ≠ `0`:
        - Ensure (`strideCount` − `1`) × `stride` + `lastStride` ≤ `bufferSize`.
  3. Increment `this`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)
  4. Let `bindingState` be a snapshot of `this`’s current state.
  5. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Draw `instanceCount` instances, starting with instance `firstInstance`, of
primitives consisting of `indexCount` indexed vertices, starting with index `firstIndex` from vertex `baseVertex`,
with the states from `bindingState` and `renderState`.
Note:
    WebGPU applications should never use index data with indices out of bounds of any
    bound vertex buffer that has [GPUVertexStepMode](https://gpuweb.github.io#enumdef-gpuvertexstepmode)["vertex"](https://gpuweb.github.io#dom-gpuvertexstepmode-vertex)[invalid memory reference](https://gpuweb.github.io/gpuweb/wgsl/#invalid-memory-reference).
- `drawIndirect(indirectBuffer, indirectOffset)`
- Draws primitives using parameters read from a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[§ 23.2 Rendering](https://gpuweb.github.io#rendering-operations) for the detailed specification.
The indirect draw parameters encoded in the buffer must be a tightly
packed block of **four 32-bit unsigned integer values (16 bytes total)**, given in the same
order as the arguments for [draw()](https://gpuweb.github.io#dom-gpurendercommandsmixin-draw)
let drawIndirectParameters= new Uint32Array( 4 ); drawIndirectParameters\[ 0 \] = vertexCount; drawIndirectParameters\[ 1 \] = instanceCount; drawIndirectParameters\[ 2 \] = firstVertex; drawIndirectParameters\[ 3 \] = firstInstance; The value corresponding to `firstInstance` must be 0, unless the ["indirect-first-instance"](https://gpuweb.github.io#indirect-first-instance)[feature](https://gpuweb.github.io#feature) is enabled.  If the ["indirect-first-instance"](https://gpuweb.github.io#indirect-first-instance)[feature](https://gpuweb.github.io#feature) is not enabled and `firstInstance` is not zero the [drawIndirect()](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindirect)
**Called on:** [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
**Arguments:**
Arguments for the [GPURenderCommandsMixin.drawIndirect(indirectBuffer, indirectOffset)](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindirect) method.Parameter Type Nullable Optional Description `indirectBuffer`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ Buffer containing the [indirect draw parameters](https://gpuweb.github.io#indirect-draw-parameters). `indirectOffset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✘ Offset in bytes into `indirectBuffer` where the drawing data begins. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - It is [valid to draw](https://gpuweb.github.io#abstract-opdef-valid-to-draw) with `this`.
    - `indirectBuffer` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `indirectBuffer`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)
    - `indirectOffset` + sizeof([indirect draw parameters](https://gpuweb.github.io#indirect-draw-parameters)) ≤ `indirectBuffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)
    - `indirectOffset` is a multiple of 4.
  3. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) `indirectBuffer` to [\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[input](https://gpuweb.github.io#internal-usage-input).
  4. Increment `this`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)
  5. Let `bindingState` be a snapshot of `this`’s current state.
  6. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `vertexCount` be an unsigned 32-bit integer read from `indirectBuffer` at `indirectOffset` bytes.
  2. Let `instanceCount` be an unsigned 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 4) bytes.
  3. Let `firstVertex` be an unsigned 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 8) bytes.
  4. Let `firstInstance` be an unsigned 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 12) bytes.
  5. Draw `instanceCount` instances, starting with instance `firstInstance`, of
primitives consisting of `vertexCount` vertices, starting with vertex `firstVertex`,
with the states from `bindingState` and `renderState`.
- `drawIndexedIndirect(indirectBuffer, indirectOffset)`
- Draws indexed primitives using parameters read from a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[§ 23.2 Rendering](https://gpuweb.github.io#rendering-operations) for the detailed specification.
The indirect drawIndexed parameters encoded in the buffer must be a
tightly packed block of **five 32-bit values (20 bytes total)**, given in the same order as
the arguments for [drawIndexed()](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindexed)`baseVertex` is a signed 32-bit integer, and all others are unsigned 32-bit integers.
For example:
let drawIndexedIndirectParameters= new Uint32Array( 5 ); let drawIndexedIndirectParametersSigned= new Int32Array( drawIndexedIndirectParameters. buffer); drawIndexedIndirectParameters\[ 0 \] = indexCount; drawIndexedIndirectParameters\[ 1 \] = instanceCount; drawIndexedIndirectParameters\[ 2 \] = firstIndex; // baseVertex is a signed value. drawIndexedIndirectParametersSigned\[ 3 \] = baseVertex; drawIndexedIndirectParameters\[ 4 \] = firstInstance; The value corresponding to `firstInstance` must be 0, unless the ["indirect-first-instance"](https://gpuweb.github.io#indirect-first-instance)[feature](https://gpuweb.github.io#feature) is enabled.  If the ["indirect-first-instance"](https://gpuweb.github.io#indirect-first-instance)[feature](https://gpuweb.github.io#feature) is not enabled and `firstInstance` is not zero the [drawIndexedIndirect()](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindexedindirect)
**Called on:** [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
**Arguments:**
Arguments for the [GPURenderCommandsMixin.drawIndexedIndirect(indirectBuffer, indirectOffset)](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawindexedindirect) method.Parameter Type Nullable Optional Description `indirectBuffer`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ Buffer containing the [indirect drawIndexed parameters](https://gpuweb.github.io#indirect-drawindexed-parameters). `indirectOffset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✘ Offset in bytes into `indirectBuffer` where the drawing data begins. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - It is [valid to draw indexed](https://gpuweb.github.io#abstract-opdef-valid-to-draw-indexed) with `this`.
    - `indirectBuffer` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `indirectBuffer`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)
    - `indirectOffset` + sizeof([indirect drawIndexed parameters](https://gpuweb.github.io#indirect-drawindexed-parameters)) ≤ `indirectBuffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)
    - `indirectOffset` is a multiple of 4.
  3. [Add](https://gpuweb.github.io#abstract-opdef-usage-scope-add) `indirectBuffer` to [\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[input](https://gpuweb.github.io#internal-usage-input).
  4. Increment `this`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)
  5. Let `bindingState` be a snapshot of `this`’s current state.
  6. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `indexCount` be an unsigned 32-bit integer read from `indirectBuffer` at `indirectOffset` bytes.
  2. Let `instanceCount` be an unsigned 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 4) bytes.
  3. Let `firstIndex` be an unsigned 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 8) bytes.
  4. Let `baseVertex` be a signed 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 12) bytes.
  5. Let `firstInstance` be an unsigned 32-bit integer read from `indirectBuffer` at
(`indirectOffset` + 16) bytes.
  6. Draw `instanceCount` instances, starting with instance `firstInstance`, of
primitives consisting of `indexCount` indexed vertices, starting with index `firstIndex` from vertex `baseVertex`,
with the states from `bindingState` and `renderState`.

[GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
`encoder`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. If any of the following conditions are unsatisfied, return `false`:
  - [Validate encoder bind groups](https://gpuweb.github.io#abstract-opdef-validate-encoder-bind-groups)(`encoder`, `encoder`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)`true`.
  - [Validate immediate data](https://gpuweb.github.io#abstract-opdef-validate-immediate-data)(`encoder`, `encoder`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)`true`.
  - Let `pipelineDescriptor` be `encoder`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)
  - For each [GPUIndex32](https://gpuweb.github.io#typedefdef-gpuindex32)`slot` `0` to `pipelineDescriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)[size](https://infra.spec.whatwg.org/#list-size):
    - If `pipelineDescriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)`slot`\] is not `null`, `encoder`.[\[\[vertex\_buffers\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffers-slot)[contain](https://infra.spec.whatwg.org/#map-exists) `slot`.
  - Validate [maxBindGroupsPlusVertexBuffers](https://gpuweb.github.io#dom-supported-limits-maxbindgroupsplusvertexbuffers)
    1. Let `bindGroupSpaceUsed` be
(the maximum key in `encoder`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)
    2. Let `vertexBufferSpaceUsed` be
(the maximum key in `encoder`.[\[\[vertex\_buffers\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffers-slot)
    3. `bindGroupSpaceUsed` + `vertexBufferSpaceUsed` must be ≤ `encoder`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxBindGroupsPlusVertexBuffers](https://gpuweb.github.io#dom-supported-limits-maxbindgroupsplusvertexbuffers)
Otherwise, return `true`.

[GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)
`encoder`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. If any of the following conditions are unsatisfied, return `false`:
  - It must be [valid to draw](https://gpuweb.github.io#abstract-opdef-valid-to-draw) with `encoder`.
  - `encoder`.[\[\[index\_buffer\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer-slot)`null`.
  - Let `topology` be `encoder`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[topology](https://gpuweb.github.io#dom-gpuprimitivestate-topology)
  - If `topology` is ["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)["triangle-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-strip)
    - `encoder`.[\[\[index\_format\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_format-slot)`encoder`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[stripIndexFormat](https://gpuweb.github.io#dom-gpuprimitivestate-stripindexformat)
Otherwise, return `true`.

#### 17.2.2. Rasterization state

The [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)

- `setViewport(x, y, width, height, minDepth, maxDepth)`
- Sets the viewport used during the rasterization stage to linearly map from
  [normalized device coordinates](https://gpuweb.github.io#ndc) to [viewport coordinates](https://gpuweb.github.io#viewport-coordinates).
**Called on:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
`this`. **Arguments:**
Arguments for the [GPURenderPassEncoder.setViewport(x, y, width, height, minDepth, maxDepth)](https://gpuweb.github.io#dom-gpurenderpassencoder-setviewport) method.Parameter Type Nullable Optional Description `x`
[float](https://webidl.spec.whatwg.org/#idl-float)
✘ ✘ Minimum X value of the viewport in pixels. `y`
[float](https://webidl.spec.whatwg.org/#idl-float)
✘ ✘ Minimum Y value of the viewport in pixels. `width`
[float](https://webidl.spec.whatwg.org/#idl-float)
✘ ✘ Width of the viewport in pixels. `height`
[float](https://webidl.spec.whatwg.org/#idl-float)
✘ ✘ Height of the viewport in pixels. `minDepth`
[float](https://webidl.spec.whatwg.org/#idl-float)
✘ ✘ Minimum depth value of the viewport. `maxDepth`
[float](https://webidl.spec.whatwg.org/#idl-float)
✘ ✘ Maximum depth value of the viewport. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. Let `maxViewportRange` be `this`.[limits](https://gpuweb.github.io#dom-gpudevice-limits)[maxTextureDimension2D](https://gpuweb.github.io#dom-gpusupportedlimits-maxtexturedimension2d)`2`.
  3. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `x` ≥ -`maxViewportRange`
    - `y` ≥ -`maxViewportRange`
    - `0` ≤ `width` ≤ `this`.[limits](https://gpuweb.github.io#dom-gpudevice-limits)[maxTextureDimension2D](https://gpuweb.github.io#dom-gpusupportedlimits-maxtexturedimension2d)
    - `0` ≤ `height` ≤ `this`.[limits](https://gpuweb.github.io#dom-gpudevice-limits)[maxTextureDimension2D](https://gpuweb.github.io#dom-gpusupportedlimits-maxtexturedimension2d)
    - `x` + `width` ≤ `maxViewportRange` − `1`
    - `y` + `height` ≤ `maxViewportRange` − `1`
    - `0.0` ≤ `minDepth` ≤ `1.0`
    - `0.0` ≤ `maxDepth` ≤ `1.0`
    - `minDepth` ≤ `maxDepth`
  4. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Round `x`, `y`, `width`, and `height` to some uniform precision, no less precise than integer rounding.
  2. Set `renderState`.[\[\[viewport\]\]](https://gpuweb.github.io#dom-renderstate-viewport-slot)`x`, `y`, `width`, `height`, `minDepth`, and `maxDepth`.
- `setScissorRect(x, y, width, height)`
- Sets the scissor rectangle used during the rasterization stage.
After transformation into [viewport coordinates](https://gpuweb.github.io#viewport-coordinates) any fragments which fall outside the scissor
rectangle will be discarded.
**Called on:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
`this`. **Arguments:**
Arguments for the [GPURenderPassEncoder.setScissorRect(x, y, width, height)](https://gpuweb.github.io#dom-gpurenderpassencoder-setscissorrect) method.Parameter Type Nullable Optional Description `x`
[GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate)
✘ ✘ Minimum X value of the scissor rectangle in pixels. `y`
[GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate)
✘ ✘ Minimum Y value of the scissor rectangle in pixels. `width`
[GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate)
✘ ✘ Width of the scissor rectangle in pixels. `height`
[GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate)
✘ ✘ Height of the scissor rectangle in pixels. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `x`+`width` ≤ `this`.[\[\[attachment\_size\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-attachment_size-slot)
    - `y`+`height` ≤ `this`.[\[\[attachment\_size\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-attachment_size-slot)
  3. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Set `renderState`.[\[\[scissorRect\]\]](https://gpuweb.github.io#dom-renderstate-scissorrect-slot)`x`, `y`, `width`, and `height`.
- `setBlendConstant(color)`
- Sets the constant blend color and alpha values used with ["constant"](https://gpuweb.github.io#dom-gpublendfactor-constant)["one-minus-constant"](https://gpuweb.github.io#dom-gpublendfactor-one-minus-constant)[GPUBlendFactor](https://gpuweb.github.io#enumdef-gpublendfactor)
**Called on:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
**Arguments:**
Arguments for the [GPURenderPassEncoder.setBlendConstant(color)](https://gpuweb.github.io#dom-gpurenderpassencoder-setblendconstant) method.Parameter Type Nullable Optional Description `color`
[GPUColor](https://gpuweb.github.io#typedefdef-gpucolor)
✘ ✘ The color to use when blending. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUColor shape](https://gpuweb.github.io#abstract-opdef-validate-gpucolor-shape)(`color`).
  2. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Set `renderState`.[\[\[blendConstant\]\]](https://gpuweb.github.io#dom-renderstate-blendconstant-slot)`color`.
- `setStencilReference(reference)`
- Sets the [\[\[stencilReference\]\]](https://gpuweb.github.io#dom-renderstate-stencilreference-slot)["replace"](https://gpuweb.github.io#dom-gpustenciloperation-replace)[GPUStencilOperation](https://gpuweb.github.io#enumdef-gpustenciloperation)
**Called on:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
**Arguments:**
Arguments for the [GPURenderPassEncoder.setStencilReference(reference)](https://gpuweb.github.io#dom-gpurenderpassencoder-setstencilreference) method.Parameter Type Nullable Optional Description `reference`
[GPUStencilValue](https://gpuweb.github.io#typedefdef-gpustencilvalue)
✘ ✘ The new stencil reference value. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Set `renderState`.[\[\[stencilReference\]\]](https://gpuweb.github.io#dom-renderstate-stencilreference-slot)`reference`.

#### 17.2.3. Queries

- `beginOcclusionQuery(queryIndex)`
- **Called on:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
`this`. **Arguments:**
Arguments for the [GPURenderPassEncoder.beginOcclusionQuery(queryIndex)](https://gpuweb.github.io#dom-gpurenderpassencoder-beginocclusionquery) method.Parameter Type Nullable Optional Description `queryIndex`
[GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
✘ ✘ The index of the query in the query set. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `this`.[\[\[occlusion\_query\_set\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_set-slot)`null`.
    - `queryIndex` \< `this`.[\[\[occlusion\_query\_set\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_set-slot)[count](https://gpuweb.github.io#dom-gpuqueryset-count)
    - The query at same `queryIndex` must not have been previously written to in this pass.
    - `this`.[\[\[occlusion\_query\_active\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_active-slot)`false`.
  3. Set `this`.[\[\[occlusion\_query\_active\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_active-slot)`true`.
  4. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Set `renderState`.[\[\[occlusionQueryIndex\]\]](https://gpuweb.github.io#dom-renderstate-occlusionqueryindex-slot)`queryIndex`.
- `endOcclusionQuery()`
- **Called on:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
**Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - `this`.[\[\[occlusion\_query\_active\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_active-slot)`true`.
  3. Set `this`.[\[\[occlusion\_query\_active\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_active-slot)`false`.
  4. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `passingFragments` be non-zero if any fragment samples passed all per-fragment
tests since the corresponding [beginOcclusionQuery()](https://gpuweb.github.io#dom-gpurenderpassencoder-beginocclusionquery)
Note: If no draw calls occurred, `passingFragments` is zero.
  2. Write `passingFragments` into `this`.[\[\[occlusion\_query\_set\]\]](https://gpuweb.github.io#dom-gpurenderpassencoder-occlusion_query_set-slot)`renderState`.[\[\[occlusionQueryIndex\]\]](https://gpuweb.github.io#dom-renderstate-occlusionqueryindex-slot)

#### 17.2.4. Bundles

- `executeBundles(bundles)`
- Executes the commands previously recorded into the given [GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)
When a [GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)[GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)
Note: The state is cleared, not restored to the previous state.
This occurs even if zero [GPURenderBundles](https://gpuweb.github.io#gpurenderbundle)
**Called on:** [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
**Arguments:**
Arguments for the [GPURenderPassEncoder.executeBundles(bundles)](https://gpuweb.github.io#dom-gpurenderpassencoder-executebundles) method.Parameter Type Nullable Optional Description `bundles`
[sequence](https://webidl.spec.whatwg.org/#idl-sequence)<[GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)>
✘ ✘ List of render bundles to execute. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. [Validate the encoder state](https://gpuweb.github.io#abstract-opdef-validate-the-encoder-state) of `this`. If it returns false, return.
  2. If any of the following conditions are unsatisfied, [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `this` and return.
    - For each `bundle` in `bundles`:
      - `bundle` must be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
      - `this`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-layout-slot)`bundle`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpurenderbundle-layout-slot)
      - If `this`.[\[\[depthReadOnly\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-depthreadonly-slot)`bundle`.[\[\[depthReadOnly\]\]](https://gpuweb.github.io#dom-gpurenderbundle-depthreadonly-slot)
      - If `this`.[\[\[stencilReadOnly\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-stencilreadonly-slot)`bundle`.[\[\[stencilReadOnly\]\]](https://gpuweb.github.io#dom-gpurenderbundle-stencilreadonly-slot)
  3. For each `bundle` in `bundles`:
    1. Increment `this`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)`bundle`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurenderbundle-drawcount-slot)
    2. [Merge](https://gpuweb.github.io#abstract-opdef-usage-scope-merge) `bundle`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)`this`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)
    3. [Extend](https://infra.spec.whatwg.org/#set-extend) `this`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-used_bind_groups-slot)`bundle`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpurenderbundle-used_bind_groups-slot)
    4. [Enqueue a render command](https://gpuweb.github.io#abstract-opdef-enqueue-a-render-command) on `this` which issues the following steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) with `renderState` when executed:
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
      1. Execute each command in `bundle`.[\[\[command\_list\]\]](https://gpuweb.github.io#dom-gpurenderbundle-command_list-slot)`renderState`.
Note: `renderState` cannot be changed by executing render bundles. Binding state was
already captured at bundle encoding time, and so isn’t used when executing bundles.
  4. [Reset the render pass binding state](https://gpuweb.github.io#abstract-opdef-reset-the-render-pass-binding-state) of `this`.

[GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)
`encoder`run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

1. [Clear](https://infra.spec.whatwg.org/#map-clear) `encoder`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)
2. Set each entry of `encoder`.[\[\[immediate\_slots\_set\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-immediate_slots_set-slot)`false`.
3. Set `encoder`.[\[\[pipeline\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-pipeline-slot)`null`.
4. Set `encoder`.[\[\[index\_buffer\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer-slot)`null`.
5. [Clear](https://infra.spec.whatwg.org/#map-clear) `encoder`.[\[\[vertex\_buffers\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffers-slot)

## 18. Bundles

A bundle is a partial, limited pass that is encoded once and can then be executed multiple times as part of future pass encoders without expiring after use like typical command buffers. This can reduce the overhead of encoding and submission of commands which are issued repeatedly without changing.

### 18.1. `GPURenderBundle`

\[Exposed Window ,Worker ),SecureContext interface GPURenderBundle GPURenderBundle includes GPUObjectBase 

- `[[command_list]]`, of type [list](https://infra.spec.whatwg.org/#list)\<[GPU command](https://gpuweb.github.io#gpu-command)>
- A [list](https://infra.spec.whatwg.org/#list) of [GPU commands](https://gpuweb.github.io#gpu-command) to be submitted to the [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)
- `[[used_bind_groups]]`, of type [set](https://infra.spec.whatwg.org/#ordered-set)\<[GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
- A [set](https://infra.spec.whatwg.org/#ordered-set) of all [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)
- `[[usage scope]]`, of type [usage scope](https://gpuweb.github.io#usage-scope), initially empty
- The [usage scope](https://gpuweb.github.io#usage-scope) for this render bundle, stored for later merging into the [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[executeBundles()](https://gpuweb.github.io#dom-gpurenderpassencoder-executebundles)
- `[[layout]]`, of type [GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)
- The layout of the render bundle.
- `[[depthReadOnly]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- If `true`, indicates that the depth component is not modified by executing this render bundle.
- `[[stencilReadOnly]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
- If `true`, indicates that the stencil component is not modified by executing this render bundle.
- `[[drawCount]]`, of type [GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
- The number of draw commands in this [GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)

#### 18.1.1. Render Bundle Creation

dictionary GPURenderBundleDescriptor GPUObjectDescriptorBase 

\[Exposed Window ,Worker ),SecureContext interface GPURenderBundleEncoder GPURenderBundle finish optional GPURenderBundleDescriptor descriptor GPURenderBundleEncoder includes GPUObjectBase GPURenderBundleEncoder includes GPUCommandsMixin GPURenderBundleEncoder includes GPUDebugCommandsMixin GPURenderBundleEncoder includes GPUBindingCommandsMixin GPURenderBundleEncoder includes GPURenderCommandsMixin 

- `createRenderBundleEncoder(descriptor)`
- Creates a [GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.createRenderBundleEncoder(descriptor)](https://gpuweb.github.io#dom-gpudevice-createrenderbundleencoder) method.Parameter Type Nullable Optional Description `descriptor`
[GPURenderBundleEncoderDescriptor](https://gpuweb.github.io#dictdef-gpurenderbundleencoderdescriptor)
✘ ✘ Description of the [GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)
**Returns:** [GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of each non-`null` element of `descriptor`.[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  2. If `descriptor`.[depthStencilFormat](https://gpuweb.github.io#dom-gpurenderpasslayout-depthstencilformat)[provided](https://infra.spec.whatwg.org/#map-exists):
    1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of `descriptor`.[depthStencilFormat](https://gpuweb.github.io#dom-gpurenderpasslayout-depthstencilformat)`this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  3. Let `e` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)`descriptor`).
  4. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  5. Return `e`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. If any of the following conditions are unsatisfied
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `e` and return.
    - `this` must not be [lost](https://gpuweb.github.io#abstract-opdef-invalid).
    - `descriptor`.[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)[size](https://infra.spec.whatwg.org/#list-size) must be ≤ `this`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxColorAttachments](https://gpuweb.github.io#dom-supported-limits-maxcolorattachments)
    - For each non-`null` `colorFormat` in `descriptor`.[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)
      - `colorFormat` must be a [color renderable format](https://gpuweb.github.io#color-renderable-format).
    - [Calculating color attachment bytes per sample](https://gpuweb.github.io#abstract-opdef-calculating-color-attachment-bytes-per-sample)(`descriptor`.[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)`this`.[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)[maxColorAttachmentBytesPerSample](https://gpuweb.github.io#dom-supported-limits-maxcolorattachmentbytespersample)
    - If `descriptor`.[depthStencilFormat](https://gpuweb.github.io#dom-gpurenderpasslayout-depthstencilformat)[provided](https://infra.spec.whatwg.org/#map-exists):
      - `descriptor`.[depthStencilFormat](https://gpuweb.github.io#dom-gpurenderpasslayout-depthstencilformat)[depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format).
    - There must exist at least one attachment, either: 
      - A non-`null` value in `descriptor`.[colorFormats](https://gpuweb.github.io#dom-gpurenderpasslayout-colorformats)
      - A `descriptor`.[depthStencilFormat](https://gpuweb.github.io#dom-gpurenderpasslayout-depthstencilformat)
  2. Set `e`.[\[\[layout\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-layout-slot)`descriptor`’s included [GPURenderPassLayout](https://gpuweb.github.io#dictdef-gpurenderpasslayout)
  3. Set `e`.[\[\[depthReadOnly\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-depthreadonly-slot)`descriptor`.[depthReadOnly](https://gpuweb.github.io#dom-gpurenderbundleencoderdescriptor-depthreadonly)
  4. Set `e`.[\[\[stencilReadOnly\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-stencilreadonly-slot)`descriptor`.[stencilReadOnly](https://gpuweb.github.io#dom-gpurenderbundleencoderdescriptor-stencilreadonly)
  5. Set `e`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[open](https://gpuweb.github.io#encoder-state-open)".
  6. Set `e`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)

#### 18.1.2. Encoding

dictionary GPURenderBundleEncoderDescriptor GPURenderPassLayout boolean depthReadOnly false ;boolean stencilReadOnly false ; };

- `depthReadOnly`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- If `true`, indicates that the render bundle does not modify the depth component of the [GPURenderPassDepthStencilAttachment](https://gpuweb.github.io#dictdef-gpurenderpassdepthstencilattachment)
See [read-only depth-stencil](https://gpuweb.github.io#read-only-depth-stencil).
- `stencilReadOnly`,  of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean), defaulting to `false`
- If `true`, indicates that the render bundle does not modify the stencil component of the [GPURenderPassDepthStencilAttachment](https://gpuweb.github.io#dictdef-gpurenderpassdepthstencilattachment)
See [read-only depth-stencil](https://gpuweb.github.io#read-only-depth-stencil).

#### 18.1.3. Finalization

- `finish(descriptor)`
- Completes recording of the render bundle commands sequence. **Called on:** [GPURenderBundleEncoder](https://gpuweb.github.io#gpurenderbundleencoder)
**Arguments:**
Arguments for the [GPURenderBundleEncoder.finish(descriptor)](https://gpuweb.github.io#dom-gpurenderbundleencoder-finish) method.Parameter Type Nullable Optional Description `descriptor` [GPURenderBundleDescriptor](https://gpuweb.github.io#dictdef-gpurenderbundledescriptor)
✘ ✔ **Returns:** [GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `renderBundle` be a new [GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)
  2. Issue the `finish steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  3. Return `renderBundle`.
[Device timeline](https://gpuweb.github.io#device-timeline) `finish steps`:
  1. Let `validationSucceeded` be `true` if all of the following requirements are met, and `false` otherwise.
    - `this` must be [valid](https://gpuweb.github.io#abstract-opdef-valid).
    - `this`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)[usage scope validation](https://gpuweb.github.io#usage-scope-validation).
    - `this`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[open](https://gpuweb.github.io#encoder-state-open)".
    - `this`.[\[\[debug\_group\_stack\]\]](https://gpuweb.github.io#dom-gpudebugcommandsmixin-debug_group_stack-slot)[be empty](https://infra.spec.whatwg.org/#list-is-empty).
  2. Set `this`.[\[\[state\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-state-slot)[ended](https://gpuweb.github.io#encoder-state-ended)".
  3. If `validationSucceeded` is `false`, then:
    1. [Generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error).
    2. Return an [invalidated](https://gpuweb.github.io#abstract-opdef-invalidate) [GPURenderBundle](https://gpuweb.github.io#gpurenderbundle)
  4. Set `renderBundle`.[\[\[command\_list\]\]](https://gpuweb.github.io#dom-gpurenderbundle-command_list-slot)`this`.[\[\[commands\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-commands-slot)
  5. Set `renderBundle`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpurenderbundle-used_bind_groups-slot)`this`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandsmixin-used_bind_groups-slot)
  6. Set `renderBundle`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurenderbundle-usage-scope-slot)`this`.[\[\[usage scope\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-usage-scope-slot)
  7. Set `renderBundle`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurenderbundle-drawcount-slot)`this`.[\[\[drawCount\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-drawcount-slot)

## 19. Queues

### 19.1. `GPUQueueDescriptor`

[GPUQueueDescriptor](https://gpuweb.github.io#gpuqueuedescriptor)

dictionary GPUQueueDescriptor GPUObjectDescriptorBase 

### 19.2. `GPUQueue`

\[Exposed Window ,Worker ),SecureContext interface GPUQueue undefined submit sequence GPUCommandBuffer commandBuffers Promise undefined onSubmittedWorkDone undefined writeBuffer GPUBuffer buffer GPUSize64 bufferOffset AllowSharedBufferSource data optional GPUSize64 dataOffset optional GPUSize64 size undefined writeTexture GPUTexelCopyTextureInfo destination AllowSharedBufferSource data GPUTexelCopyBufferLayout dataLayout GPUExtent3D size undefined copyExternalImageToTexture GPUCopyExternalImageSourceInfo source GPUCopyExternalImageDestInfo destination GPUExtent3D copySize GPUQueue includes GPUObjectBase 

[GPUQueue](https://gpuweb.github.io#gpuqueue)

- `writeBuffer(buffer, bufferOffset, data, dataOffset, size)`
- Issues a write operation of the provided data into a [GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUQueue](https://gpuweb.github.io#gpuqueue)
`this`. **Arguments:**
Arguments for the [GPUQueue.writeBuffer(buffer, bufferOffset, data, dataOffset, size)](https://gpuweb.github.io#dom-gpuqueue-writebuffer) method.Parameter Type Nullable Optional Description `buffer`
[GPUBuffer](https://gpuweb.github.io#gpubuffer)
✘ ✘ The buffer to write to. `bufferOffset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✘ Offset in bytes into `buffer` to begin writing at. `data`
[AllowSharedBufferSource](https://webidl.spec.whatwg.org/#AllowSharedBufferSource)
✘ ✘ Data to write into `buffer`. `dataOffset`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Offset in into `data` to begin writing from. Given in elements if `data` is a `TypedArray` and bytes otherwise. `size`
[GPUSize64](https://gpuweb.github.io#typedefdef-gpusize64)
✘ ✔ Size of content to write from `data` to `buffer`. Given in elements if `data` is a `TypedArray` and bytes otherwise. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If `data` is an [ArrayBuffer](https://webidl.spec.whatwg.org/#idl-ArrayBuffer)[DataView](https://webidl.spec.whatwg.org/#idl-DataView)`data` is a TypedArray; let the element type be the type of the TypedArray.
  2. Let `dataSize` be the size of `data`, in elements.
  3. If `size` is missing,
let `contentsSize` be `dataSize` − `dataOffset`.
Otherwise, let `contentsSize` be `size`.
  4. If any of the following conditions are unsatisfied,
throw an [OperationError](https://webidl.spec.whatwg.org/#operationerror)
    - `contentsSize` ≥ 0.
    - `dataOffset` + `contentsSize` ≤ `dataSize`.
    - `contentsSize`, converted to bytes, is a multiple of 4 bytes.
  5. Let `dataContents` be [a copy of the bytes held by the buffer source](https://webidl.spec.whatwg.org/#dfn-get-buffer-source-copy) `data`.
  6. Let `contents` be the `contentsSize` elements of `dataContents` starting at
an offset of `dataOffset` elements.
  7. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. If any of the following conditions are unsatisfied,
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error) and return.
    - `buffer` is [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - `buffer`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[available](https://gpuweb.github.io#gpubuffer-internal-state-available)".
    - `buffer`.[usage](https://gpuweb.github.io#dom-gpubuffer-usage)[COPY\_DST](https://gpuweb.github.io#dom-gpubufferusage-copy_dst)
    - `bufferOffset`, converted to bytes, is a multiple of 4 bytes.
    - `bufferOffset` + `contentsSize`, converted to bytes, ≤ `buffer`.[size](https://gpuweb.github.io#dom-gpubuffer-size)
  2. Issue the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) of `this`.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Write `contents` into `buffer` starting at `bufferOffset`.
- `writeTexture(destination, data, dataLayout, size)`
- Issues a write operation of the provided data into a [GPUTexture](https://gpuweb.github.io#gputexture)
**Called on:** [GPUQueue](https://gpuweb.github.io#gpuqueue)
`this`. **Arguments:**
Arguments for the [GPUQueue.writeTexture(destination, data, dataLayout, size)](https://gpuweb.github.io#dom-gpuqueue-writetexture) method.Parameter Type Nullable Optional Description `destination`
[GPUTexelCopyTextureInfo](https://gpuweb.github.io#gputexelcopytextureinfo)
✘ ✘ The [texture subresource](https://gpuweb.github.io#texture-subresources) and origin to write to. `data`
[AllowSharedBufferSource](https://webidl.spec.whatwg.org/#AllowSharedBufferSource)
✘ ✘ Data to write into `destination`. `dataLayout`
[GPUTexelCopyBufferLayout](https://gpuweb.github.io#gputexelcopybufferlayout)
✘ ✘ Layout of the content in `data`. `size`
[GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
✘ ✘ Extents of the content to write from `data` to `destination`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUOrigin3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuorigin3d-shape)(`destination`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  2. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUExtent3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuextent3d-shape)(`size`).
  3. Let `dataBytes` be [a copy of the bytes held by the buffer source](https://webidl.spec.whatwg.org/#dfn-get-buffer-source-copy) `data`.
Note: This is described as copying all of `data` to the device timeline,
but in practice `data` could be much larger than necessary.
Implementations should optimize by copying only the necessary bytes.
  4. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Let `aligned` be `false`.
  2. Let `dataLength` be `dataBytes`.[length](https://infra.spec.whatwg.org/#byte-sequence-length).
  3. If any of the following conditions are unsatisfied,
[generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error) and return.
    - `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[\[\[destroyed\]\]](https://gpuweb.github.io#dom-gputexture-destroyed-slot)`false`.
    - [validating texture buffer copy](https://gpuweb.github.io#abstract-opdef-validating-texture-buffer-copy)(`destination`, `dataLayout`, `dataLength`, `size`, [COPY\_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)`aligned`) returns `true`.
Note: unlike
    [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[copyBufferToTexture()](https://gpuweb.github.io#dom-gpucommandencoder-copybuffertotexture)`dataLayout`.[bytesPerRow](https://gpuweb.github.io#dom-gputexelcopybufferlayout-bytesperrow)`dataLayout`.[offset](https://gpuweb.github.io#dom-gputexelcopybufferlayout-offset)
  4. Issue the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) of `this`.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. Let `blockWidth` be the [texel block width](https://gpuweb.github.io#texel-block-width) of `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  2. Let `blockHeight` be the [texel block height](https://gpuweb.github.io#texel-block-height) of `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  3. Let `dstOrigin` be `destination`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  4. Let `dstBlockOriginX` be (`dstOrigin`.[x](https://gpuweb.github.io#gpuorigin3d-x) ÷ `blockWidth`).
  5. Let `dstBlockOriginY` be (`dstOrigin`.[y](https://gpuweb.github.io#gpuorigin3d-y) ÷ `blockHeight`).
  6. Let `blockColumns` be (`copySize`.[width](https://gpuweb.github.io#gpuextent3d-width) ÷ `blockWidth`).
  7. Let `blockRows` be (`copySize`.[height](https://gpuweb.github.io#gpuextent3d-height) ÷ `blockHeight`).
  8. [Assert](https://infra.spec.whatwg.org/#assert) that `dstBlockOriginX`, `dstBlockOriginY`, `blockColumns`, and `blockRows` are integers.
  9. For each `z` in the range \[0, `copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) − 1\]:
    1. Let `dstSubregion` be [texture copy sub-region](https://gpuweb.github.io#abstract-opdef-texture-copy-sub-region) (`z` + `dstOrigin`.[z](https://gpuweb.github.io#gpuorigin3d-z)) of `destination`.
    2. For each `y` in the range \[0, `blockRows` − 1\]:
      1. For each `x` in the range \[0, `blockColumns` − 1\]:
        1. Let `blockOffset` be the [texel block byte offset](https://gpuweb.github.io#abstract-opdef-texel-block-byte-offset) of `dataLayout` for (`x`, `y`, `z`) of `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
        2. Set [texel block](https://gpuweb.github.io#texel-block) (`dstBlockOriginX` + `x`, `dstBlockOriginY` + `y`) of `dstSubregion` to be an [equivalent texel representation](https://gpuweb.github.io#equivalent-texel-representation) to the [texel block](https://gpuweb.github.io#texel-block) described by `dataBytes` at offset `blockOffset`.
- `copyExternalImageToTexture(source, destination, copySize)`
- Issues a copy operation of the contents of a platform image/canvas into the destination texture. This operation performs [color encoding](https://gpuweb.github.io#color-space-conversions) into the destination
encoding according to the parameters of [GPUCopyExternalImageDestInfo](https://gpuweb.github.io#gpucopyexternalimagedestinfo)
Copying into a `-srgb` texture results in the same texture bytes, not the same decoded
values, as copying into the corresponding non-`-srgb` format.
Thus, after a copy operation, sampling the destination texture has
different results depending on whether its format is `-srgb`, all else unchanged.
NOTE:When copying from a`"webgl"`/`"webgl2"` context canvas, the [WebGL Drawing Buffer](https://www.khronos.org/registry/webgl/specs/latest/1.0/#THE_DRAWING_BUFFER) may be not exist during certain points in the
    frame presentation cycle (after the image has been moved to the compositor
    for display). To avoid this, either:
  - Issue [copyExternalImageToTexture()](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture)[task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task) with
WebGL rendering operation, to ensure the copy occurs before the WebGL
canvas is presented.
  - If not possible, set the `preserveDrawingBuffer` option in [WebGLContextAttributes](https://www.khronos.org/registry/webgl/specs/latest/1.0/#WEBGLCONTEXTATTRIBUTES)`true`, so that the drawing buffer will
still contain a copy of the frame contents after they’ve been presented.
Note, this extra copy may have a performance cost.
**Called on:** [GPUQueue](https://gpuweb.github.io#gpuqueue)
`this`. **Arguments:**
Arguments for the [GPUQueue.copyExternalImageToTexture(source, destination, copySize)](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture) method.Parameter Type Nullable Optional Description `source`
[GPUCopyExternalImageSourceInfo](https://gpuweb.github.io#gpucopyexternalimagesourceinfo)
✘ ✘ source image and origin to copy to `destination`. `destination`
[GPUCopyExternalImageDestInfo](https://gpuweb.github.io#gpucopyexternalimagedestinfo)
✘ ✘ The [texture subresource](https://gpuweb.github.io#texture-subresources) and origin to write to, and its encoding metadata. `copySize`
[GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
✘ ✘ Extents of the content to write from `source` to `destination`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUOrigin2D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuorigin2d-shape)(`source`.[origin](https://gpuweb.github.io#dom-gpucopyexternalimagesourceinfo-origin)
  2. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUOrigin3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuorigin3d-shape)(`destination`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  3. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [validate GPUExtent3D shape](https://gpuweb.github.io#abstract-opdef-validate-gpuextent3d-shape)(`copySize`).
  4. Let `sourceImage` be `source`.[source](https://gpuweb.github.io#dom-gpucopyexternalimagesourceinfo-source)
  5. If `sourceImage` [is not origin-clean](https://html.spec.whatwg.org/multipage/canvas.html#the-image-argument-is-not-origin-clean),
throw a [SecurityError](https://webidl.spec.whatwg.org/#securityerror)
  6. If any of the following requirements are unmet, throw an [OperationError](https://webidl.spec.whatwg.org/#operationerror)
    - `source`.`origin`.[x](https://gpuweb.github.io#gpuorigin2d-x) + `copySize`.[width](https://gpuweb.github.io#gpuextent3d-width) must be ≤ the width of `sourceImage`.
    - `source`.`origin`.[y](https://gpuweb.github.io#gpuorigin2d-y) + `copySize`.[height](https://gpuweb.github.io#gpuextent3d-height) must be ≤ the height of `sourceImage`.
    - `copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) must be ≤ 1.
  7. Let `usability` be [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [check the usability of the image argument](https://html.spec.whatwg.org/multipage/canvas.html#check-the-usability-of-the-image-argument)(`source`).
  8. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Let `texture` be `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)
  2. If any of the following requirements are unmet, [generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error) and return.
    - `usability` must be `good`.
    - `texture`.[\[\[destroyed\]\]](https://gpuweb.github.io#dom-gputexture-destroyed-slot)`false`.
    - `texture` must be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`.
    - [validating GPUTexelCopyTextureInfo](https://gpuweb.github.io#abstract-opdef-validating-gputexelcopytextureinfo)(destination, copySize) must return `true`.
    - `texture`.[usage](https://gpuweb.github.io#dom-gputexture-usage)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[COPY\_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)
    - `texture`.[dimension](https://gpuweb.github.io#dom-gputexture-dimension)["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)
    - `texture`.[sampleCount](https://gpuweb.github.io#dom-gputexture-samplecount)
    - `texture`.[format](https://gpuweb.github.io#dom-gputexture-format)[plain color format](https://gpuweb.github.io#plain-color-formats) supporting [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)`unorm`/`unorm-srgb` or `float`/`ufloat` format (not `snorm`, `uint`, or `sint`).
  3. If `copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) is > 0, issue the subsequent
steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) of `this`.
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. [Assert](https://infra.spec.whatwg.org/#assert) that the [texel block width](https://gpuweb.github.io#texel-block-width) of `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)[texel block height](https://gpuweb.github.io#texel-block-height) of `destination`.[texture](https://gpuweb.github.io#dom-gputexelcopytextureinfo-texture)`copySize`.[depthOrArrayLayers](https://gpuweb.github.io#gpuextent3d-depthorarraylayers) is 1.
  2. Let `srcOrigin` be `source`.[origin](https://gpuweb.github.io#dom-gpucopyexternalimagesourceinfo-origin)
  3. Let `dstOrigin` be `destination`.[origin](https://gpuweb.github.io#dom-gputexelcopytextureinfo-origin)
  4. Let `dstSubregion` be [texture copy sub-region](https://gpuweb.github.io#abstract-opdef-texture-copy-sub-region) (`dstOrigin`.[z](https://gpuweb.github.io#gpuorigin3d-z)) of `destination`.
  5. For each `y` in the range \[0, `copySize`.[height](https://gpuweb.github.io#gpuextent3d-height) − 1\]:
    1. Let `srcY` be `y` if `source`.[flipY](https://gpuweb.github.io#dom-gpucopyexternalimagesourceinfo-flipy)`false` and
(`copySize`.[height](https://gpuweb.github.io#gpuextent3d-height) − 1 − `y`) otherwise.
    2. For each `x` in the range \[0, `copySize`.[width](https://gpuweb.github.io#gpuextent3d-width) − 1\]:
      1. Let `srcColor` be the [color-managed color value](https://gpuweb.github.io#color-spaces) of the pixel at
(`srcOrigin`.[x](https://gpuweb.github.io#gpuorigin2d-x) + `x`, `srcOrigin`.[y](https://gpuweb.github.io#gpuorigin2d-y) + `srcY`) of `source`.[source](https://gpuweb.github.io#dom-gpucopyexternalimagesourceinfo-source)
      2. Let `dstColor` be the numeric RGBA value resulting from applying any [color encoding](https://gpuweb.github.io#color-space-conversions) required by `destination`.[colorSpace](https://gpuweb.github.io#dom-gpucopyexternalimagedestinfo-colorspace)`destination`.[premultipliedAlpha](https://gpuweb.github.io#dom-gpucopyexternalimagedestinfo-premultipliedalpha)`srcColor`.
      3. If `texture`.[format](https://gpuweb.github.io#dom-gputexture-format)`-srgb` format:
        1. Set `dstColor` to the result of applying the sRGB non-linear-to-linear conversion to it.
Note:
This cancels out the sRGB linear-to-non-linear conversion that occurs
when writing an `-srgb` format in the next step, so that precision
from an sRGB-like input image is not lost and the *linear* color values
of the original image can be read from the texture
(as is generally the purpose of using `-srgb` formats).
      4. Set [texel block](https://gpuweb.github.io#texel-block) (`dstOrigin`.[x](https://gpuweb.github.io#gpuorigin3d-x) + `x`, `dstOrigin`.[y](https://gpuweb.github.io#gpuorigin3d-y) + `y`) of `dstSubregion` to an [equivalent texel representation](https://gpuweb.github.io#equivalent-texel-representation) of `dstColor`.
- `submit(commandBuffers)`
- Schedules the execution of the command buffers by the GPU on this queue. Submitted command buffers cannot be used again. **Called on:** [GPUQueue](https://gpuweb.github.io#gpuqueue)
**Arguments:**
Arguments for the [GPUQueue.submit(commandBuffers)](https://gpuweb.github.io#dom-gpuqueue-submit) method.Parameter Type Nullable Optional Description `commandBuffers`
[sequence](https://webidl.spec.whatwg.org/#idl-sequence)<[GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)>
✘ ✘ **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`:
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. If any of the following requirements are unmet, [generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) each [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)`commandBuffers` and return.
    - Every [GPUCommandBuffer](https://gpuweb.github.io#gpucommandbuffer)`commandBuffers` must be unique.
    - For each `commandBuffer` in `commandBuffers`:
      - `commandBuffer` must be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `this`
      - For each `bindGroup` in `commandBuffer`.[\[\[used\_bind\_groups\]\]](https://gpuweb.github.io#dom-gpucommandbuffer-used_bind_groups-slot)
        - For each [GPUBindingResource](https://gpuweb.github.io#typedefdef-gpubindingresource)`bindGroup`, if the resource type is:
          - [GPUBuffer](https://gpuweb.github.io#gpubuffer)
`b`
          - `b`.[\[\[internal state\]\]](https://gpuweb.github.io#dom-gpubuffer-internal-state-slot)[available](https://gpuweb.github.io#gpubuffer-internal-state-available)".
          - [GPUTexture](https://gpuweb.github.io#gputexture)
`t`
          - `t`.[\[\[destroyed\]\]](https://gpuweb.github.io#dom-gputexture-destroyed-slot)`false`.
          - [GPUExternalTexture](https://gpuweb.github.io#gpuexternaltexture)
`et`
          - `et`.[\[\[expired\]\]](https://gpuweb.github.io#dom-gpuexternaltexture-expired-slot)`false`.
          - [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
`qs`
          - `qs`.[\[\[destroyed\]\]](https://gpuweb.github.io#dom-gpuqueryset-destroyed-slot)`false`.
Note:
For occlusion queries, the [occlusionQuerySet](https://gpuweb.github.io#dom-gpurenderpassdescriptor-occlusionqueryset)[beginRenderPass()](https://gpuweb.github.io#dom-gpucommandencoder-beginrenderpass)[beginOcclusionQuery()](https://gpuweb.github.io#dom-gpurenderpassencoder-beginocclusionquery)
  2. For each `commandBuffer` in `commandBuffers`:
    1. [Invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `commandBuffer`.
  3. Issue the subsequent steps on the [Queue timeline](https://gpuweb.github.io#queue-timeline) of `this`:
[Queue timeline](https://gpuweb.github.io#queue-timeline) steps:
  1. For each `commandBuffer` in `commandBuffers`:
    1. Execute each command in `commandBuffer`.[\[\[command\_list\]\]](https://gpuweb.github.io#dom-gpucommandbuffer-command_list-slot)
- `onSubmittedWorkDone()`
- Returns a [Promise](https://webidl.spec.whatwg.org/#idl-promise)
Resolution of this [Promise](https://webidl.spec.whatwg.org/#idl-promise)[mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)[GPUBuffer](https://gpuweb.github.io#gpubuffer)
**Called on:** [GPUQueue](https://gpuweb.github.io#gpuqueue)
`this`. [Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `contentTimeline` be the current [Content timeline](https://gpuweb.github.io#content-timeline).
  2. Let `promise` be [a new promise](https://webidl.spec.whatwg.org/#a-new-promise).
  3. Issue the `synchronization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  4. Return `promise`.
[Device timeline](https://gpuweb.github.io#device-timeline) `synchronization steps`:
  1. Let `event` occur upon the completion of
all currently-enqueued operations.
  2. [Listen for timeline event](https://gpuweb.github.io#abstract-opdef-listen-for-timeline-event) `event` on `this`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)`contentTimeline`.
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise`.

## 20. Queries

### 20.1. `GPUQuerySet`

\[Exposed Window ,Worker ),SecureContext interface GPUQuerySet undefined destroy readonly attribute GPUQueryType type readonly attribute GPUSize32Out count GPUQuerySet includes GPUObjectBase 

[GPUQuerySet](https://gpuweb.github.io#gpuqueryset)[immutable properties](https://gpuweb.github.io#immutable-property):

- `type`,  of type [GPUQueryType](https://gpuweb.github.io#enumdef-gpuquerytype), readonly
- The type of the queries managed by this [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
- `count`,  of type [GPUSize32Out](https://gpuweb.github.io#typedefdef-gpusize32out), readonly
- The number of queries managed by this [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)

[GPUQuerySet](https://gpuweb.github.io#gpuqueryset)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[destroyed]]`, of type [boolean](https://webidl.spec.whatwg.org/#idl-boolean)
`false`
- If the query set is destroyed, it can no longer be used in any operation, and its underlying memory can be freed.

#### 20.1.1. QuerySet Creation

A [GPUQuerySetDescriptor](https://gpuweb.github.io#dictdef-gpuquerysetdescriptor)[GPUQuerySet](https://gpuweb.github.io#gpuqueryset)

dictionary GPUQuerySetDescriptor GPUObjectDescriptorBase required GPUQueryType type required GPUSize32 count 

- `type`,  of type [GPUQueryType](https://gpuweb.github.io#enumdef-gpuquerytype)
- The type of queries managed by [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
- `count`,  of type [GPUSize32](https://gpuweb.github.io#typedefdef-gpusize32)
- The number of queries managed by [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)

- `createQuerySet(descriptor)`
- Creates a [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
**Arguments:**
Arguments for the [GPUDevice.createQuerySet(descriptor)](https://gpuweb.github.io#dom-gpudevice-createqueryset) method.Parameter Type Nullable Optional Description `descriptor` [GPUQuerySetDescriptor](https://gpuweb.github.io#dictdef-gpuquerysetdescriptor)
✘ ✘ Description of the [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
**Returns:** [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If `descriptor`.[type](https://gpuweb.github.io#dom-gpuquerysetdescriptor-type)["timestamp"](https://gpuweb.github.io#dom-gpuquerytype-timestamp)["timestamp-query"](https://gpuweb.github.io#timestamp-query)[enabled for](https://gpuweb.github.io#enabled-for) `this`:
    1. Throw a [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)
  2. Let `q` be [!](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [create a new WebGPU object](https://gpuweb.github.io#abstract-opdef-create-a-new-webgpu-object)(`this`, [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)`descriptor`).
  3. Set `q`.[type](https://gpuweb.github.io#dom-gpuqueryset-type)`descriptor`.[type](https://gpuweb.github.io#dom-gpuquerysetdescriptor-type)
  4. Set `q`.[count](https://gpuweb.github.io#dom-gpuqueryset-count)`descriptor`.[count](https://gpuweb.github.io#dom-gpuquerysetdescriptor-count)
  5. Issue the `initialization steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  6. Return `q`.
[Device timeline](https://gpuweb.github.io#device-timeline) `initialization steps`:
  1. If any of the following requirements are unmet, [generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `q` and return.
  2. Create a device allocation for `q` where each entry in the query set is zero.
If the allocation fails without side-effects,
[generate an out-of-memory error](https://gpuweb.github.io#abstract-opdef-generate-an-out-of-memory-error), [invalidate](https://gpuweb.github.io#abstract-opdef-invalidate) `q`, and return.

[Creating a](https://gpuweb.github.io#example-efcca31b)

[GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
const querySet= gpuDevice. createQuerySet({ type: 'occlusion' , count: 32 }); 

#### 20.1.2. Query Set Destruction

An application that no longer requires a [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)[destroy()](https://gpuweb.github.io#dom-gpuqueryset-destroy)

[GPUQuerySet](https://gpuweb.github.io#gpuqueryset)

- `destroy()`
- Destroys the [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
**Called on:** [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)
`this`. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [device timeline](https://gpuweb.github.io#device-timeline).
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Set `this`.[\[\[destroyed\]\]](https://gpuweb.github.io#dom-gpuqueryset-destroyed-slot)`true`.

### 20.2. QueryType

enum GPUQueryType "occlusion" "timestamp" 

### 20.3. Occlusion Query

Occlusion query is only available on render passes, to query the number of fragment samples that pass all the per-fragment tests for a set of drawing commands, including scissor, sample mask, alpha to coverage, stencil, and depth tests. Any non-zero result value for the query indicates that at least one sample passed the tests and reached the output merging stage of the render pipeline, 0 indicates that no samples passed the tests.

When beginning a render pass, [GPURenderPassDescriptor](https://gpuweb.github.io#dictdef-gpurenderpassdescriptor)[occlusionQuerySet](https://gpuweb.github.io#dom-gpurenderpassdescriptor-occlusionqueryset)[beginOcclusionQuery()](https://gpuweb.github.io#dom-gpurenderpassencoder-beginocclusionquery)[endOcclusionQuery()](https://gpuweb.github.io#dom-gpurenderpassencoder-endocclusionquery)[GPUBuffer](https://gpuweb.github.io#gpubuffer)[64-bit unsigned integer](https://gpuweb.github.io/gpuweb/wgsl/#64-bit-integer) by [GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[resolveQuerySet()](https://gpuweb.github.io#dom-gpucommandencoder-resolvequeryset)

### 20.4. Timestamp Query

Timestamp queries allow applications to write timestamps to a [GPUQuerySet](https://gpuweb.github.io#gpuqueryset)

and then resolve timestamp values (in nanoseconds as a [64-bit unsigned integer](https://gpuweb.github.io/gpuweb/wgsl/#64-bit-integer)) into
a [GPUBuffer](https://gpuweb.github.io#gpubuffer)[GPUCommandEncoder](https://gpuweb.github.io#gpucommandencoder)[resolveQuerySet()](https://gpuweb.github.io#dom-gpucommandencoder-resolvequeryset)

Timestamp values are [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined).
Applications must handle arbitrary timestamp results, and should not be written in such a way that unexpected
timestamps cause an application failure.

Note: The physical device may reset the timestamp counter occasionally, which can result in unexpected values such as negative deltas from one timestamp to the next. These instances should be rare, and these data points can safely be discarded.

 
[Timestamp queries are implemented using high-resolution timers (see](https://infra.spec.whatwg.org/#tracking-vector) [§ 2.1.7.2 Device/queue-timeline timing](https://gpuweb.github.io#security-timing-device)).
To mitigate security and privacy concerns, their precision must be reduced:

[queue timeline](https://gpuweb.github.io#queue-timeline)steps:

- Let `fineTimestamp` be the current timestamp value of the current [queue timeline](https://gpuweb.github.io#queue-timeline),
in nanoseconds, relative to an [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined) point in the past.
- Return the result of calling [coarsen time](https://w3c.github.io/hr-time/#dfn-coarsen-time) on `fineTimestamp` with `crossOriginIsolatedCapability` set to `false`.

Note: Cross-origin isolation never applies to the [device timeline](https://gpuweb.github.io#device-timeline) or
    [queue timeline](https://gpuweb.github.io#queue-timeline), so `crossOriginIsolatedCapability` is never set to `true`.

`device`,

`timestampWrites`)

**Arguments:**

- [GPUDevice](https://gpuweb.github.io#gpudevice)`device`
- `(` [GPUComputePassTimestampWrites](https://gpuweb.github.io#dictdef-gpucomputepasstimestampwrites)[GPURenderPassTimestampWrites](https://gpuweb.github.io#dictdef-gpurenderpasstimestampwrites)`timestampWrites`

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Return `true` if the following requirements are met, and `false` if not:
  - ["timestamp-query"](https://gpuweb.github.io#timestamp-query)[enabled for](https://gpuweb.github.io#enabled-for) `device`.
  - `timestampWrites`.`querySet` must be [valid to use with](https://gpuweb.github.io#abstract-opdef-valid-to-use-with) `device`.
  - `timestampWrites`.`querySet`.[type](https://gpuweb.github.io#dom-gpuqueryset-type)["timestamp"](https://gpuweb.github.io#dom-gpuquerytype-timestamp)
  - Of the write index members in `timestampWrites` (`beginningOfPassWriteIndex`, `endOfPassWriteIndex`):

## 21. Canvas Rendering

### 21.1. [HTMLCanvasElement.getContext()](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-getcontext)

A [GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)[created](https://gpuweb.github.io#abstract-opdef-create-a-webgpu-context-on-a-canvas)
via the [getContext()](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-getcontext)[HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)`'webgpu'` as its `contextType` argument.

[Get a](https://gpuweb.github.io#example-c3c7f9c6)

[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)[HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)
const canvas= document. createElement( 'canvas' ); const context= canvas. getContext( 'webgpu' ); 

Unlike WebGL or 2D context creation, the second argument of
[HTMLCanvasElement.getContext()](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-getcontext)[OffscreenCanvas.getContext()](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-getcontext)`options`, is ignored.
Instead, use [GPUCanvasContext.configure()](https://gpuweb.github.io#dom-gpucanvascontext-configure)

[HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)[OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)
`canvas`, run the following

[content timeline](https://gpuweb.github.io#content-timeline)steps:

1. Let `context` be a new [GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
2. Set `context`.[canvas](https://gpuweb.github.io#dom-gpucanvascontext-canvas)`canvas`.
3. [Replace the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer) of `context`.
4. Return `context`.

Note: User agents should consider issuing developer-visible warnings when
    an ignored `options` argument is provided when calling `getContext()`
    to get a WebGPU canvas context.

### 21.2. GPUCanvasContext

\[Exposed Window ,Worker ),SecureContext interface GPUCanvasContext readonly attribute (HTMLCanvasElement or OffscreenCanvas canvas undefined configure GPUCanvasConfiguration configuration undefined unconfigure GPUCanvasConfiguration getConfiguration GPUTexture getCurrentTexture 

[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)[content timeline properties](https://gpuweb.github.io#content-timeline-property):

- `canvas`,  of type `(HTMLCanvasElement or OffscreenCanvas)`, readonly
- The canvas this context was created from.
- `[[configuration]]`, of type [GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)
`null`
- The options this context is currently configured with. `null` if the context has not been configured or has been [unconfigured](https://gpuweb.github.io#dom-gpucanvascontext-unconfigure)
- `[[textureDescriptor]]`, of type [GPUTextureDescriptor](https://gpuweb.github.io#gputexturedescriptor)
`null`
- The currently configured texture descriptor, derived from the
[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)
`null` if the context has not been configured or has been [unconfigured](https://gpuweb.github.io#dom-gpucanvascontext-unconfigure)
- `[[drawingBuffer]]`, an image, initially
a transparent black image with the same size as the canvas
- The drawing buffer is the working-copy image data of the canvas.
It is exposed as writable by [\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
The drawing buffer is used to [get a copy of the image contents of a context](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context), which
occurs when the canvas is displayed or otherwise read. It may be transparent, even if [\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)[alphaMode](https://gpuweb.github.io#dom-gpucanvasconfiguration-alphamode)["opaque"](https://gpuweb.github.io#dom-gpucanvasalphamode-opaque)[alphaMode](https://gpuweb.github.io#dom-gpucanvasconfiguration-alphamode)[get a copy of the image contents of a context](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context)" algorithm.
The drawing buffer outlives the [\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)[Replace the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer).
Any time the drawing buffer is read, implementations must ensure that all previously
submitted work (e.g. queue submissions) have completed writing to it via
[\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)
- `[[currentTexture]]`, of type [GPUTexture](https://gpuweb.github.io#gputexture)
`null`
- The [GPUTexture](https://gpuweb.github.io#gputexture)[\[\[drawingBuffer\]\]](https://gpuweb.github.io#dom-gpucanvascontext-drawingbuffer-slot)[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)`null`, then returns it.
In the steady-state of a visible canvas, any changes to the drawing buffer made through the
currentTexture get presented when [updating the rendering of a WebGPU canvas](https://gpuweb.github.io#abstract-opdef-updating-the-rendering-of-a-webgpu-canvas).
At or before that point, the texture is also destroyed
and [\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)`null`, signalling that
a new one is to be created by the next call to [getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
[Destroying](https://gpuweb.github.io#dom-gputexture-destroy)[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
[Expire the current texture](https://gpuweb.github.io#abstract-opdef-expire-the-current-texture) sets the currentTexture to `null`.
It is called by [configure()](https://gpuweb.github.io#dom-gpucanvascontext-configure)[transferToImageBitmap()](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-transfertoimagebitmap)
- `[[lastPresentedImage]]`, of type `(readonly image)?`, initially `null`
- The image most recently presented for this canvas in "[updating the rendering of a WebGPU canvas](https://gpuweb.github.io#abstract-opdef-updating-the-rendering-of-a-webgpu-canvas)".
If the device is lost or destroyed, this image **may** be used as a fallback in
"[get a copy of the image contents of a context](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context)" in order to prevent the canvas from going blank.
Note: This property only needs to exist in implementations which implement the fallback, which is optional.

[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)

- `configure(configuration)`
- Configures the context for this canvas.
This clears the drawing buffer to transparent black (in [Replace the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer)).
See [getConfiguration()](https://gpuweb.github.io#dom-gpucanvascontext-getconfiguration)[feature detection](https://gpuweb.github.io#feature-detection).
**Called on:** [GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`this`. **Arguments:**
Arguments for the [GPUCanvasContext.configure(configuration)](https://gpuweb.github.io#dom-gpucanvascontext-configure) method.Parameter Type Nullable Optional Description `configuration`
[GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)
✘ ✘ Desired configuration for the context. **Returns:** undefined
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `device` be `configuration`.[device](https://gpuweb.github.io#dom-gpucanvasconfiguration-device)
  2. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of `configuration`.[format](https://gpuweb.github.io#dom-gpucanvasconfiguration-format)`device`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  3. [?](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands) [Validate texture format required features](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) of each element of `configuration`.[viewFormats](https://gpuweb.github.io#dom-gpucanvasconfiguration-viewformats)`device`.[\[\[device\]\]](https://gpuweb.github.io#dom-gpuobjectbase-device-slot)
  4. If [Supported context formats](https://gpuweb.github.io#supported-context-formats) does not [contain](https://infra.spec.whatwg.org/#list-contain) `configuration`.[format](https://gpuweb.github.io#dom-gpucanvasconfiguration-format)[TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)
  5. If `configuration`.[usage](https://gpuweb.github.io#dom-gpucanvasconfiguration-usage)[TRANSIENT\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-transient_attachment)[TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)
  6. Let `descriptor` be the [GPUTextureDescriptor for the canvas and configuration](https://gpuweb.github.io#abstract-opdef-gputexturedescriptor-for-the-canvas-and-configuration)(`this`.[canvas](https://gpuweb.github.io#dom-gpucanvascontext-canvas)`configuration`).
  7. Set `this`.[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)`configuration`.
Note:
This exposes only the members defined in an implementation’s definition of
[GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)[feature detection](https://gpuweb.github.io#feature-detection).
  8. Set `this`.[\[\[textureDescriptor\]\]](https://gpuweb.github.io#dom-gpucanvascontext-texturedescriptor-slot)`descriptor`.
  9. [Replace the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer) of `this`.
  10. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `device`.
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. If any of the following requirements are unmet, [generate a validation error](https://gpuweb.github.io#abstract-opdef-generate-a-validation-error) and return.
    - [validating GPUTextureDescriptor](https://gpuweb.github.io#abstract-opdef-validating-gputexturedescriptor)(`device`, `descriptor`)
must return true.
Note: This early validation remains valid until the next
[configure()](https://gpuweb.github.io#dom-gpucanvascontext-configure)**except** for
validation of the [size](https://gpuweb.github.io#dom-gputexturedescriptor-size)
- `unconfigure()`
- Removes the context configuration. Destroys any textures produced while configured. **Called on:** [GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`this`. **Returns:** undefined
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Set `this`.[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)`null`.
  2. Set `this`.[\[\[textureDescriptor\]\]](https://gpuweb.github.io#dom-gpucanvascontext-texturedescriptor-slot)`null`.
  3. [Replace the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer) of `this`.
- `getConfiguration()`
- Returns the context configuration, or `null` if the context is not configured.
Note:
This method exists primarily for [feature detection](https://gpuweb.github.io#feature-detection) of members (and sub-members) of [GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)
**Called on:** [GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`this`. **Returns:** [GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)`null`
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `configuration` be a copy of `this`.[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)
  2. Return `configuration`.
- `getCurrentTexture()`
- Get the [GPUTexture](https://gpuweb.github.io#gputexture)[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
NOTE:An application**should** call [getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
The expiry task (defined below) is optional to implement.
    Even if implemented, task source priority is not normatively defined, so may happen as
    early as the next task, or as late as after all other task sources are empty
    (see [automatic expiry task source](https://gpuweb.github.io#automatic-expiry-task-source)).
    Expiry is only guaranteed when a visible canvas is displayed
    ([updating the rendering of a WebGPU canvas](https://gpuweb.github.io#abstract-opdef-updating-the-rendering-of-a-webgpu-canvas)) and in other
    callers of "[Expire the current texture](https://gpuweb.github.io#abstract-opdef-expire-the-current-texture)".
**Called on:** [GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`this`. **Returns:** [GPUTexture](https://gpuweb.github.io#gputexture)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. If `this`.[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)`null`,
throw an [InvalidStateError](https://webidl.spec.whatwg.org/#invalidstateerror)
  2. [Assert](https://infra.spec.whatwg.org/#assert) `this`.[\[\[textureDescriptor\]\]](https://gpuweb.github.io#dom-gpucanvascontext-texturedescriptor-slot)`null`.
  3. Let `device` be `this`.[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)[device](https://gpuweb.github.io#dom-gpucanvasconfiguration-device)
  4. If `this`.[\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)`null`:
    1. [Replace the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer) of `this`.
    2. Set `this`.[\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)`device`.[createTexture()](https://gpuweb.github.io#dom-gpudevice-createtexture)`this`.[\[\[textureDescriptor\]\]](https://gpuweb.github.io#dom-gpucanvascontext-texturedescriptor-slot)[GPUTexture](https://gpuweb.github.io#gputexture)`this`.[\[\[drawingBuffer\]\]](https://gpuweb.github.io#dom-gpucanvascontext-drawingbuffer-slot)
Note:
If the texture can’t be created (e.g. due to validation failure or out-of-memory),
this generates and error and returns an [invalidated](https://gpuweb.github.io#abstract-opdef-invalidate) [GPUTexture](https://gpuweb.github.io#gputexture)[configure()](https://gpuweb.github.io#dom-gpucanvascontext-configure)**must not** skip this redundant validation.
  5. **Optionally**, [queue an automatic expiry task](https://gpuweb.github.io#abstract-opdef-queue-an-automatic-expiry-task) with device `device` and the following steps:
    1. [Expire the current texture](https://gpuweb.github.io#abstract-opdef-expire-the-current-texture) of `this`.
Note: If this already happened when
[updating the rendering of a WebGPU canvas](https://gpuweb.github.io#abstract-opdef-updating-the-rendering-of-a-webgpu-canvas), it has no effect.
  6. Return `this`.[\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)
Note: The same [GPUTexture](https://gpuweb.github.io#gputexture)[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)[Expire the current texture](https://gpuweb.github.io#abstract-opdef-expire-the-current-texture)"
runs, even if that [GPUTexture](https://gpuweb.github.io#gputexture)

**Arguments:**

- `context`: the [GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)

**Returns:** image contents

[Content timeline](https://gpuweb.github.io#content-timeline) steps:

1. Let `snapshot` be a transparent black image of the same size as `context`.[canvas](https://gpuweb.github.io#dom-gpucanvascontext-canvas)
2. Let `configuration` be `context`.[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)
3. If `configuration` is `null`:
  1. Return `snapshot`.
Note: The configuration will be `null` if the context has not been
configured or has been [unconfigured](https://gpuweb.github.io#dom-gpucanvascontext-unconfigure)
4. Ensure that all submitted work items (e.g. queue submissions) have
completed writing to the image (via `context`.[\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)
5. If `configuration`.[device](https://gpuweb.github.io#dom-gpucanvasconfiguration-device)[valid](https://w3c.github.io/i18n-glossary/#dfn-valid):
  1. Set `snapshot` to a copy of the `context`.[\[\[drawingBuffer\]\]](https://gpuweb.github.io#dom-gpucanvascontext-drawingbuffer-slot)
Otherwise, if `context`.[\[\[lastPresentedImage\]\]](https://gpuweb.github.io#dom-gpucanvascontext-lastpresentedimage-slot)`null`:
  1. **Optionally**, set `snapshot` to a copy of `context`.[\[\[lastPresentedImage\]\]](https://gpuweb.github.io#dom-gpucanvascontext-lastpresentedimage-slot)
Note:
This is optional because the [\[\[lastPresentedImage\]\]](https://gpuweb.github.io#dom-gpucanvascontext-lastpresentedimage-slot)
6. Let `alphaMode` be `configuration`.[alphaMode](https://gpuweb.github.io#dom-gpucanvasconfiguration-alphamode)
7. If `alphaMode` is ["opaque"](https://gpuweb.github.io#dom-gpucanvasalphamode-opaque)
  1. Clear the alpha channel of `snapshot` to 1.0.
Note:
If the [\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)[Expire the current texture](https://gpuweb.github.io#abstract-opdef-expire-the-current-texture)"), the alpha channel is unobservable,
and implementations may clear the alpha channel in-place.
  2. Tag `snapshot` as being opaque.
Otherwise: 
  1. Tag `snapshot` with `alphaMode`.
8. Tag `snapshot` with the [colorSpace](https://gpuweb.github.io#dom-gpucanvasconfiguration-colorspace)[toneMapping](https://gpuweb.github.io#dom-gpucanvasconfiguration-tonemapping)`configuration`.
9. Return `snapshot`.

[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`context`, run the following

[content timeline](https://gpuweb.github.io#content-timeline)steps:

1. [Expire the current texture](https://gpuweb.github.io#abstract-opdef-expire-the-current-texture) of `context`.
2. Let `configuration` be `context`.[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)
3. Set `context`.[\[\[drawingBuffer\]\]](https://gpuweb.github.io#dom-gpucanvascontext-drawingbuffer-slot)`context`.[canvas](https://gpuweb.github.io#dom-gpucanvascontext-canvas)
  - If `configuration` is null, the drawing buffer is tagged with the color space ["srgb"](https://html.spec.whatwg.org/multipage/canvas.html#dom-predefinedcolorspace-srgb)
  - If not, the drawing buffer has the specified
`configuration`.[format](https://gpuweb.github.io#dom-gpucanvasconfiguration-format)`configuration`.[colorSpace](https://gpuweb.github.io#dom-gpucanvasconfiguration-colorspace)`configuration`.[toneMapping](https://gpuweb.github.io#dom-gpucanvasconfiguration-tonemapping)
Note: `configuration`.[alphaMode](https://gpuweb.github.io#dom-gpucanvasconfiguration-alphamode)[get a copy of the image contents of a context](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context)".
NOTE:A newly replaced drawing buffer image behaves as if it is cleared to transparent black, but, like after["discard"](https://gpuweb.github.io#dom-gpustoreop-discard)
Note: This will often be a no-op, if the drawing buffer is already cleared and has the correct configuration.

[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`context`, run the following

[content timeline](https://gpuweb.github.io#content-timeline)steps:

1. If `context`.[\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)`null`:
  1. Call `context`.[\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)[destroy()](https://gpuweb.github.io#dom-gputexture-destroy)`context`.[\[\[drawingBuffer\]\]](https://gpuweb.github.io#dom-gpucanvascontext-drawingbuffer-slot)
  2. Set `context`.[\[\[currentTexture\]\]](https://gpuweb.github.io#dom-gpucanvascontext-currenttexture-slot)`null`.

### 21.3. HTML Specification Hooks

The following algorithms "hook" into algorithms in the HTML specification, and must run at the specified points.

[HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)[OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`context`, run the following

[content timeline](https://gpuweb.github.io#content-timeline)steps:

1. Return [a copy of the image contents](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context) of `context`.

- When an [HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)
  - Including when the canvas is the [placeholder canvas element](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas-placeholder) of an [OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)
- When [transferToImageBitmap()](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-transfertoimagebitmap)[ImageBitmap](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#imagebitmap)[transferToImageBitmap from WebGPU](https://gpuweb.github.io#abstract-opdef-transfertoimagebitmap-from-webgpu).)
- When WebGPU canvas contents are read using other Web APIs, like
[drawImage()](https://html.spec.whatwg.org/multipage/canvas.html#dom-context-2d-drawimage)`texImage2D()`, `texSubImage2D()`, [toDataURL()](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-todataurl)[toBlob()](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-toblob)

If [alphaMode](https://gpuweb.github.io#dom-gpucanvasconfiguration-alphamode)["opaque"](https://gpuweb.github.io#dom-gpucanvasalphamode-opaque)

If an application needs a canvas only for interop (not presentation), avoid
        ["opaque"](https://gpuweb.github.io#dom-gpucanvasalphamode-opaque)

[HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)[OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)
[placeholder canvas element](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas-placeholder)) with a

[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`context`, which occurs before getting the canvas’s image contents, in the following sub-steps of the

[event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model):

- "update the rendering or user interface of that `Document`"
- "update the rendering of that dedicated worker"

Note:
    Service and Shared workers do not have "update the rendering" steps
    because they cannot render to user-visible canvases.
    [requestAnimationFrame()](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#dom-animationframeprovider-requestanimationframe)[ServiceWorkerGlobalScope](https://w3c.github.io/ServiceWorker/#serviceworkerglobalscope)[SharedWorkerGlobalScope](https://html.spec.whatwg.org/multipage/workers.html#sharedworkerglobalscope)[OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)[transferControlToOffscreen()](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-transfercontroltooffscreen)[cannot be sent to these workers](https://github.com/whatwg/html/issues/10112).

Run the following [content timeline](https://gpuweb.github.io#content-timeline) steps:

1. [Expire the current texture](https://gpuweb.github.io#abstract-opdef-expire-the-current-texture) of `context`.
Note: If this already happened in the task queued by
[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
2. Set `context`.[\[\[lastPresentedImage\]\]](https://gpuweb.github.io#dom-gpucanvascontext-lastpresentedimage-slot)`context`.[\[\[drawingBuffer\]\]](https://gpuweb.github.io#dom-gpucanvascontext-drawingbuffer-slot)
Note: This is just a reference, not a copy; the drawing buffer’s contents can’t change in-place after the current texture has expired.

Note:
    This does not happen for standalone [OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)`new OffscreenCanvas()`).

When [transferToImageBitmap()](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-transfertoimagebitmap)[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)`context`, after creating an [ImageBitmap](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#imagebitmap)[content timeline](https://gpuweb.github.io#content-timeline) steps:

1. [Replace the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer) of `context`.

Note: This makes [transferToImageBitmap()](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-transfertoimagebitmap)

- The [update the canvas size](https://gpuweb.github.io#abstract-opdef-update-the-canvas-size) algorithm.

### 21.4. GPUCanvasConfiguration

The supported context formats are the [set](https://infra.spec.whatwg.org/#ordered-set) of [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)["bgra8unorm"](https://gpuweb.github.io#dom-gputextureformat-bgra8unorm)["rgba8unorm"](https://gpuweb.github.io#dom-gputextureformat-rgba8unorm)["rgba16float"](https://gpuweb.github.io#dom-gputextureformat-rgba16float)[GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)[format](https://gpuweb.github.io#dom-gpucanvasconfiguration-format)[GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)[device](https://gpuweb.github.io#dom-gpucanvasconfiguration-device)

Note: Canvas configuration cannot use `srgb` formats like ["bgra8unorm-srgb"](https://gpuweb.github.io#dom-gputextureformat-bgra8unorm-srgb)`srgb` equivalent (["bgra8unorm"](https://gpuweb.github.io#dom-gputextureformat-bgra8unorm)`srgb`
format in the [viewFormats](https://gpuweb.github.io#dom-gpucanvasconfiguration-viewformats)[createView()](https://gpuweb.github.io#dom-gputexture-createview)`srgb` format.

enum GPUCanvasAlphaMode "opaque" "premultiplied" enum GPUCanvasToneMappingMode "standard" "extended" dictionary GPUCanvasToneMapping GPUCanvasToneMappingMode mode dictionary GPUCanvasConfiguration required GPUDevice device required GPUTextureFormat format GPUTextureUsageFlags usage sequence GPUTextureFormat viewFormats PredefinedColorSpace colorSpace GPUCanvasToneMapping toneMapping GPUCanvasAlphaMode alphaMode 

[GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)

- `device`,  of type [GPUDevice](https://gpuweb.github.io#gpudevice)
- The [GPUDevice](https://gpuweb.github.io#gpudevice)[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
- `format`,  of type [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- The format that textures returned by [getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)[Supported context formats](https://gpuweb.github.io#supported-context-formats).
- `usage`,  of type [GPUTextureUsageFlags](https://gpuweb.github.io#typedefdef-gputextureusageflags), defaulting to `0x10`
- The usage that textures returned by [getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
- `viewFormats`,  of type sequence\<[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)>, defaulting to `[]`
- The formats that views created from textures returned by
[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
- `colorSpace`,  of type [PredefinedColorSpace](https://html.spec.whatwg.org/multipage/canvas.html#predefinedcolorspace), defaulting to `"srgb"`
- The color space that values written into textures returned by
[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
- `toneMapping`,  of type [GPUCanvasToneMapping](https://gpuweb.github.io#dictdef-gpucanvastonemapping), defaulting to `{}`
- The tone mapping determines how the content of textures returned by
[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)
NOTE:This is a required feature, but user agents might not yet implement it, effectively supporting only the default[GPUCanvasToneMapping](https://gpuweb.github.io#dictdef-gpucanvastonemapping)
**should not** exist in its implementation of [GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)
[feature detection](https://gpuweb.github.io#feature-detection) possible using [getConfiguration()](https://gpuweb.github.io#dom-gpucanvascontext-getconfiguration)
This is especially important in implementations which otherwise have HDR capabilities
    (where a [dynamic-range](https://drafts.csswg.org/mediaqueries-5/#descdef-media-dynamic-range) of [high](https://drafts.csswg.org/mediaqueries-5/#valdef-media-dynamic-range-high) would be
    exposed).
If an implementation exposes this member and a `high` dynamic range, it **should** render the
    canvas as an HDR element, not clamp values to the SDR range of the HDR display.
- `alphaMode`,  of type [GPUCanvasAlphaMode](https://gpuweb.github.io#gpucanvasalphamode), defaulting to `"opaque"`
- Determines the effect that alpha values will have on the content of textures returned by
[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)

[Configure a](https://gpuweb.github.io#example-239a6233)

[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)[GPUDevice](https://gpuweb.github.io#gpudevice)
const canvas= document. createElement( 'canvas' ); const context= canvas. getContext( 'webgpu' ); context. configure({ device: gpuDevice, format: navigator. gpu. getPreferredCanvasFormat(), }); 

[HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)[OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)
`canvas`,

[GPUCanvasConfiguration](https://gpuweb.github.io#dictdef-gpucanvasconfiguration)
`configuration`) is a

[GPUTextureDescriptor](https://gpuweb.github.io#gputexturedescriptor)
- [size](https://gpuweb.github.io#dom-gputexturedescriptor-size)`canvas`.width, `canvas`.height, 1\].
- [format](https://gpuweb.github.io#dom-gputexturedescriptor-format)`configuration`.[format](https://gpuweb.github.io#dom-gpucanvasconfiguration-format)
- [usage](https://gpuweb.github.io#dom-gputexturedescriptor-usage)`configuration`.[usage](https://gpuweb.github.io#dom-gpucanvasconfiguration-usage)
- [viewFormats](https://gpuweb.github.io#dom-gputexturedescriptor-viewformats)`configuration`.[viewFormats](https://gpuweb.github.io#dom-gpucanvasconfiguration-viewformats)

and other members set to their defaults.

`canvas`.width refers to [HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)[width](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-width)[OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)[width](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-width)`canvas`.height refers to [HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)[height](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-height)[OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)[height](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-height)

#### 21.4.1. Canvas Color Space

During presentation, the color values in the canvas are converted to the color space of the screen.

The [toneMapping](https://gpuweb.github.io#dom-gpucanvasconfiguration-tonemapping)`[0, 1]` interval in the color space of the screen.

#### 21.4.2. Canvas Context sizing

All canvas configuration is set in [configure()](https://gpuweb.github.io#dom-gpucanvascontext-configure)`width` and `height`.

Note:
Like WebGL and 2d canvas, resizing a WebGPU canvas loses the current contents of the drawing buffer.
In WebGPU, it does so by [replacing the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer).

[HTMLCanvasElement](https://html.spec.whatwg.org/multipage/canvas.html#htmlcanvaselement)[OffscreenCanvas](https://html.spec.whatwg.org/multipage/canvas.html#offscreencanvas)
`canvas`with a

[GPUCanvasContext](https://gpuweb.github.io#gpucanvascontext)
`context`has its

`width` or `height` attributes set,
    update the canvas size by running the following
    [content timeline](https://gpuweb.github.io#content-timeline)steps:

1. [Replace the drawing buffer](https://gpuweb.github.io#abstract-opdef-replace-the-drawing-buffer) of `context`.
2. Let `configuration` be `context`.[\[\[configuration\]\]](https://gpuweb.github.io#dom-gpucanvascontext-configuration-slot)
3. If `configuration` is not `null`:
  1. Set `context`.[\[\[textureDescriptor\]\]](https://gpuweb.github.io#dom-gpucanvascontext-texturedescriptor-slot)[GPUTextureDescriptor for the canvas and configuration](https://gpuweb.github.io#abstract-opdef-gputexturedescriptor-for-the-canvas-and-configuration)(`canvas`, `configuration`).

Note: This may result in a [GPUTextureDescriptor](https://gpuweb.github.io#gputexturedescriptor)[maxTextureDimension2D](https://gpuweb.github.io#dom-supported-limits-maxtexturedimension2d)[getCurrentTexture()](https://gpuweb.github.io#dom-gpucanvascontext-getcurrenttexture)

Note: This algorithm is run any time the `canvas` `width` or `height` attributes are set, even
    if their value is not changed.

### 21.5. `GPUCanvasToneMappingMode`

This enum specifies how color values are displayed to the screen.

- `"standard"`
- Color values within the standard dynamic range of the screen are unchanged, and all other color values are projected to the standard dynamic range of the screen. Note:
This projection is often accomplished by clamping color values in the color space
of the screen to the `[0, 1]` interval.
`(1.035, -0.175, -0.140)` is written to an `'srgb'` canvas. If this is presented to an sRGB screen, then this will be converted to sRGB
    (which is a no-op, because the canvas is sRGB), then projected into the display’s space.
    Using component-wise clamping, this results in the sRGB value `(1.0, 0.0, 0.0)`.
If this is presented to a Display P3 screen, then this will be converted to
    the value `(0.948, 0.106, 0.01)` in the Display P3 color space, and no
    clamping will be needed.
- `"extended"`
- Color values in the extended dynamic range of the screen are unchanged, and all other color values are projected to the extended dynamic range of the screen. Note:
This projection is often accomplished by clamping color values in the color space of
the screen to the interval of values that the screen is capable of displaying,
which may include values greater than `1`.
`(2.5, -0.15, -0.15)` is written to an `'srgb'` canvas. If this is presented to an sRGB screen that is capable of displaying values
    in the `[0, 4]` interval in sRGB space, then this will be converted to sRGB
    (which is a no-op, because the canvas is sRGB), then projected into the display’s space.
    If using component-wise clamping, this results in the sRGB value `(2.5, 0.0, 0.0)`.
If this is presented to a Display P3 screen that is capable of displaying
    values in the `[0, 2]` interval in Display P3 space, then this will be
    converted to the value `(2.3, 0.545, 0.386)` in the Display P3 color space,
    then projected into the display’s space.
    If using component-wise clamping, this results in the Display P3 value `(2.0, 0.545, 0.386)`.

### 21.6. `GPUCanvasAlphaMode`

This enum selects how the contents of the canvas will be interpreted when read, when
[displayed to the screen or used as an image source](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context)
(in drawImage, toDataURL, etc.)

Below, `src` is a value in the canvas texture, and `dst` is an image that the canvas
is being composited into (e.g. an HTML page rendering, or a 2D canvas).

- `"opaque"`
- Read RGB as opaque and ignore alpha values.
If the content is not already opaque, the alpha channel is cleared to 1.0
in "[get a copy of the image contents of a context](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context)".
- `"premultiplied"`
- Read RGBA as premultiplied: color values are premultiplied by their alpha value.
100% red at 50% alpha is `[0.5, 0, 0, 0.5]`.
If the canvas texture contains [out-of-gamut premultiplied RGBA values](https://gpuweb.github.io#out-of-gamut-premultiplied-rgba-value) at the time the
canvas contents are read, the behavior depends on whether the canvas is:
  - [used as an image source](https://gpuweb.github.io#abstract-opdef-get-a-copy-of-the-image-contents-of-a-context)
  - Values are preserved, as described in [color space conversion](https://gpuweb.github.io#color-space-conversions).
  - displayed to the screen
  - Compositing results are undefined. Note: This is true even if color space conversion would produce in-gamut values before compositing, because the intermediate format for compositing is not specified.

## 22. Errors & Debugging

During the normal course of operation of WebGPU, errors are raised via [dispatch error](https://gpuweb.github.io#abstract-opdef-dispatch-error).

After a device is [lost](https://gpuweb.github.io#lose-the-device), errors are no longer surfaced, where possible.
After this point, implementations do not need to run validation or error tracking:

- The validity of objects on the device becomes unobservable.
- [popErrorScope()](https://gpuweb.github.io#dom-gpudevice-poperrorscope)[uncapturederror](https://gpuweb.github.io#eventdef-gpudevice-uncapturederror)[GPUDevice](https://gpuweb.github.io#gpudevice)[lost](https://gpuweb.github.io#dom-gpudevice-lost)
- All operations which send a message back to the [content timeline](https://gpuweb.github.io#content-timeline) will skip their usual steps.
Most will appear to succeed, except for [mapAsync()](https://gpuweb.github.io#dom-gpubuffer-mapasync)
This makes it unobservable whether other types of operations (that don’t send messages back) actually execute or not.

### 22.1. Fatal Errors

enum GPUDeviceLostReason "unknown" "destroyed" Exposed Window ,Worker ),SecureContext interface GPUDeviceLostInfo readonly attribute GPUDeviceLostReason reason readonly attribute DOMString message partial interface GPUDevice readonly attribute Promise GPUDeviceLostInfo lost 

[GPUDevice](https://gpuweb.github.io#gpudevice)

- `lost`,  of type Promise\<[GPUDeviceLostInfo](https://gpuweb.github.io#gpudevicelostinfo)>, readonly
- A [slot-backed attribute](https://gpuweb.github.io#slot-backed-attribute) holding a promise which is created with the device, remains
pending for the lifetime of the device, then resolves when the device is lost.
Upon initialization, it is set to [a new promise](https://webidl.spec.whatwg.org/#a-new-promise).

### 22.2. `GPUError`

\[Exposed Window ,Worker ),SecureContext interface GPUError readonly attribute DOMString message 

[GPUError](https://gpuweb.github.io#gpuerror)[popErrorScope()](https://gpuweb.github.io#dom-gpudevice-poperrorscope)[uncapturederror](https://gpuweb.github.io#eventdef-gpudevice-uncapturederror)

Errors must only be generated for operations that explicitly state the conditions one may be generated under in their respective algorithms, and the subtype of error that is generated.

No errors are generated from a device which is lost.
See [§ 22 Errors & Debugging](https://gpuweb.github.io#errors-and-debugging).

Note: [GPUError](https://gpuweb.github.io#gpuerror)[message](https://gpuweb.github.io#dom-gpuerror-message)`instanceof`. Use `error.constructor.name` when it’s necessary to serialize an error (e.g. into
JSON, for a debug report).

[GPUError](https://gpuweb.github.io#gpuerror)[immutable properties](https://gpuweb.github.io#immutable-property):

- `message`,  of type [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString), readonly
- A human-readable, [localizable text](https://www.w3.org/TR/i18n-glossary/#dfn-localizable-text) message providing information about the error that
occurred.
Note: This message is generally intended for application developers to debug their applications and capture information for debug reports, not to be surfaced to end-users. Note: User agents should not include potentially machine-parsable details in this message,
such as free system memory on ["out-of-memory"](https://gpuweb.github.io#dom-gpuerrorfilter-out-of-memory)
Note: The [message](https://gpuweb.github.io#dom-gpuerror-message)[best practices for language and direction information](https://w3c.github.io/string-meta/#bp_and-reco). This includes making use of any future standards which may emerge
regarding the reporting of string language and direction metadata.
Editorial note: At the time of this writing, no language/direction recommendation is available that provides compatibility and consistency with legacy APIs, but when there is, adopt it formally.

\[Exposed Window ,Worker ),SecureContext interface GPUValidationError GPUError constructor DOMString message 

[GPUValidationError](https://gpuweb.github.io#gpuvalidationerror)[GPUError](https://gpuweb.github.io#gpuerror)[\[\[features\]\]](https://gpuweb.github.io#dom-device-features-slot)[\[\[limits\]\]](https://gpuweb.github.io#dom-device-limits-slot)

[GPUDevice](https://gpuweb.github.io#gpudevice)
`device`, run the following steps:

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `error` be a new [GPUValidationError](https://gpuweb.github.io#gpuvalidationerror)
2. [Dispatch error](https://gpuweb.github.io#abstract-opdef-dispatch-error) `error` to `device`.

\[Exposed Window ,Worker ),SecureContext interface GPUOutOfMemoryError GPUError constructor DOMString message 

[GPUOutOfMemoryError](https://gpuweb.github.io#gpuoutofmemoryerror)[GPUError](https://gpuweb.github.io#gpuerror)

[GPUDevice](https://gpuweb.github.io#gpudevice)
`device`, run the following steps:

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `error` be a new [GPUOutOfMemoryError](https://gpuweb.github.io#gpuoutofmemoryerror)
2. [Dispatch error](https://gpuweb.github.io#abstract-opdef-dispatch-error) `error` to `device`.

\[Exposed Window ,Worker ),SecureContext interface GPUInternalError GPUError constructor DOMString message 

[GPUInternalError](https://gpuweb.github.io#gpuinternalerror)[GPUError](https://gpuweb.github.io#gpuerror)[supported limits](https://gpuweb.github.io#supported-limits). The same operation may succeed on other devices or under
difference circumstances.

[GPUDevice](https://gpuweb.github.io#gpudevice)
`device`, run the following steps:

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. Let `error` be a new [GPUInternalError](https://gpuweb.github.io#gpuinternalerror)
2. [Dispatch error](https://gpuweb.github.io#abstract-opdef-dispatch-error) `error` to `device`.

### 22.3. Error Scopes

A GPU error scope captures [GPUError](https://gpuweb.github.io#gpuerror)[GPU error scope](https://gpuweb.github.io#gpu-error-scope) was current. Error scopes are used to isolate errors that occur within a set
of WebGPU calls, typically for debugging purposes or to make an operation more fault tolerant.

[GPU error scope](https://gpuweb.github.io#gpu-error-scope) has the following [device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[errors]]`, of type [list](https://infra.spec.whatwg.org/#list)\<[GPUError](https://gpuweb.github.io#gpuerror)
- The [GPUError](https://gpuweb.github.io#gpuerror)[GPU error scope](https://gpuweb.github.io#gpu-error-scope) was current.
- `[[filter]]`, of type [GPUErrorFilter](https://gpuweb.github.io#enumdef-gpuerrorfilter)
- Determines what type of [GPUError](https://gpuweb.github.io#gpuerror)[GPU error scope](https://gpuweb.github.io#gpu-error-scope) observes.

enum GPUErrorFilter "validation" "out-of-memory" "internal" partial interface GPUDevice undefined pushErrorScope GPUErrorFilter filter Promise GPUError popErrorScope 

[GPUErrorFilter](https://gpuweb.github.io#enumdef-gpuerrorfilter)[pushErrorScope()](https://gpuweb.github.io#dom-gpudevice-pusherrorscope)

- `"validation"`
- Indicates that the error scope will catch a [GPUValidationError](https://gpuweb.github.io#gpuvalidationerror)
- `"out-of-memory"`
- Indicates that the error scope will catch a [GPUOutOfMemoryError](https://gpuweb.github.io#gpuoutofmemoryerror)
- `"internal"`
- Indicates that the error scope will catch a [GPUInternalError](https://gpuweb.github.io#gpuinternalerror)

[GPUDevice](https://gpuweb.github.io#gpudevice)[device timeline properties](https://gpuweb.github.io#device-timeline-property):

- `[[errorScopeStack]]`, of type [stack](https://infra.spec.whatwg.org/#stack)\<[GPU error scope](https://gpuweb.github.io#gpu-error-scope)>
- A [stack](https://infra.spec.whatwg.org/#stack) of [GPU error scopes](https://gpuweb.github.io#gpu-error-scope) that have been pushed to the [GPUDevice](https://gpuweb.github.io#gpudevice)

[GPUError](https://gpuweb.github.io#gpuerror)
`error`and

[GPUDevice](https://gpuweb.github.io#gpudevice)
`device`is determined by issuing the following steps to the

[device timeline](https://gpuweb.github.io#device-timeline)of

`device`:

[Device timeline](https://gpuweb.github.io#device-timeline) steps:

1. If `error` is an instance of:
  - [GPUValidationError](https://gpuweb.github.io#gpuvalidationerror)
  - Let `type` be "validation".
  - [GPUOutOfMemoryError](https://gpuweb.github.io#gpuoutofmemoryerror)
  - Let `type` be "out-of-memory".
  - [GPUInternalError](https://gpuweb.github.io#gpuinternalerror)
  - Let `type` be "internal".
2. Let `scope` be the last [item](https://infra.spec.whatwg.org/#list-item) of `device`.[\[\[errorScopeStack\]\]](https://gpuweb.github.io#dom-gpudevice-errorscopestack-slot)
3. While `scope` is not `undefined`:
  1. If `scope`.[\[\[filter\]\]](https://gpuweb.github.io#dom-gpu-error-scope-filter-slot)`type`, return `scope`.
  2. Set `scope` to the previous [item](https://infra.spec.whatwg.org/#list-item) of `device`.[\[\[errorScopeStack\]\]](https://gpuweb.github.io#dom-gpudevice-errorscopestack-slot)
4. Return `undefined`.

[GPUError](https://gpuweb.github.io#gpuerror)
`error`on

[GPUDevice](https://gpuweb.github.io#gpudevice)
`device`, run the following

[device timeline](https://gpuweb.github.io#device-timeline)steps:

[Device timeline](https://gpuweb.github.io#device-timeline)steps:

Note: No errors are generated from a device which is lost.
        If this algorithm is called while
        `device` is [lost](https://gpuweb.github.io#abstract-opdef-invalid), it will not be observable to the application.
        See [§ 22 Errors & Debugging](https://gpuweb.github.io#errors-and-debugging).

1. Let `scope` be the [current error scope](https://gpuweb.github.io#abstract-opdef-current-error-scope) for `error` and `device`.
2. If `scope` is not `undefined`:
  1. [Append](https://infra.spec.whatwg.org/#list-append) `error` to `scope`.[\[\[errors\]\]](https://gpuweb.github.io#dom-gpu-error-scope-errors-slot)
  2. Return.
Otherwise, issue the following steps to the [content timeline](https://gpuweb.github.io#content-timeline):

[Content timeline](https://gpuweb.github.io#content-timeline)steps:

1. If the user agent chooses, [queue a global task for GPUDevice](https://gpuweb.github.io#abstract-opdef-queue-a-global-task-for-gpudevice) `device` with the following steps:
  1. Fire a [GPUUncapturedErrorEvent](https://gpuweb.github.io#gpuuncapturederrorevent)[uncapturederror](https://gpuweb.github.io#eventdef-gpudevice-uncapturederror)`device`, with an [error](https://gpuweb.github.io#dom-gpuuncapturederrorevent-error)`error`.

Note: After dispatching the event, user agents **should** surface uncaptured errors to
        developers, for example as warnings in the browser’s developer console, unless the event’s
        [defaultPrevented](https://dom.spec.whatwg.org/#dom-event-defaultprevented)[preventDefault()](https://dom.spec.whatwg.org/#dom-event-preventdefault)

Note: The user agent may choose to throttle or limit the number of [GPUUncapturedErrorEvent](https://gpuweb.github.io#gpuuncapturederrorevent)[GPUDevice](https://gpuweb.github.io#gpudevice)

- `pushErrorScope(filter)`
- Pushes a new [GPU error scope](https://gpuweb.github.io#gpu-error-scope) onto the [\[\[errorScopeStack\]\]](https://gpuweb.github.io#dom-gpudevice-errorscopestack-slot)`this`.
**Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. **Arguments:**
Arguments for the [GPUDevice.pushErrorScope(filter)](https://gpuweb.github.io#dom-gpudevice-pusherrorscope) method.Parameter Type Nullable Optional Description `filter`
[GPUErrorFilter](https://gpuweb.github.io#enumdef-gpuerrorfilter)
✘ ✘ Which class of errors this error scope observes. **Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Issue the subsequent steps on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
[Device timeline](https://gpuweb.github.io#device-timeline) steps:
  1. Let `scope` be a new [GPU error scope](https://gpuweb.github.io#gpu-error-scope).
  2. Set `scope`.[\[\[filter\]\]](https://gpuweb.github.io#dom-gpu-error-scope-filter-slot)`filter`.
  3. [Push](https://infra.spec.whatwg.org/#stack-push) `scope` onto `this`.[\[\[errorScopeStack\]\]](https://gpuweb.github.io#dom-gpudevice-errorscopestack-slot)
- `popErrorScope()`
- Pops a [GPU error scope](https://gpuweb.github.io#gpu-error-scope) off the [\[\[errorScopeStack\]\]](https://gpuweb.github.io#dom-gpudevice-errorscopestack-slot)`this` and resolves to **any** [GPUError](https://gpuweb.github.io#gpuerror)`null` if none.
There is no guarantee of the ordering of promise resolution. **Called on:** [GPUDevice](https://gpuweb.github.io#gpudevice)
`this`. [Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. Let `contentTimeline` be the current [Content timeline](https://gpuweb.github.io#content-timeline).
  2. Let `promise` be [a new promise](https://webidl.spec.whatwg.org/#a-new-promise).
  3. Issue the `check steps` on the [Device timeline](https://gpuweb.github.io#device-timeline) of `this`.
  4. Return `promise`.
[Device timeline](https://gpuweb.github.io#device-timeline) `check steps`:
  1. If `this` is [lost](https://gpuweb.github.io#abstract-opdef-invalid):
    1. Issue the following steps on
`contentTimeline`:
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
      1. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise` with `null`.
    2. Return.
Note: No errors are generated from a device which is lost.
See [§ 22 Errors & Debugging](https://gpuweb.github.io#errors-and-debugging).
  2. If any of the following requirements are unmet: 
    - `this`.[\[\[errorScopeStack\]\]](https://gpuweb.github.io#dom-gpudevice-errorscopestack-slot)[size](https://infra.spec.whatwg.org/#list-size) must be > 0.
Then issue the following steps on `contentTimeline` and return:
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
    1. [Reject](https://webidl.spec.whatwg.org/#reject) `promise` with an [OperationError](https://webidl.spec.whatwg.org/#operationerror)
  3. Let `scope` be the result of [popping](https://infra.spec.whatwg.org/#stack-pop) an [item](https://infra.spec.whatwg.org/#list-item) off of `this`.[\[\[errorScopeStack\]\]](https://gpuweb.github.io#dom-gpudevice-errorscopestack-slot)
  4. Let `error` be **any** one of the items in `scope`.[\[\[errors\]\]](https://gpuweb.github.io#dom-gpu-error-scope-errors-slot)`null` if there are none.
For any two errors E1 and E2 in the list, if E2 was caused by E1, E2 **should
not** be the one selected.
Note:
For example, if E1 comes from `t` = [createTexture()](https://gpuweb.github.io#dom-gpudevice-createtexture)`t`.[createView()](https://gpuweb.github.io#dom-gputexture-createview)`t` was [invalid](https://gpuweb.github.io#abstract-opdef-invalid),
E1 should be be preferred since it will be easier for a developer to understand
what went wrong.
Since both of these are [GPUValidationError](https://gpuweb.github.io#gpuvalidationerror)[message](https://gpuweb.github.io#dom-gpuerror-message)
  5. At an **unspecified point now or in the future**,
issue the subsequent steps on `contentTimeline`.
Note:
By allowing [popErrorScope()](https://gpuweb.github.io#dom-gpudevice-poperrorscope)
[Content timeline](https://gpuweb.github.io#content-timeline) steps:
  1. [Resolve](https://webidl.spec.whatwg.org/#resolve) `promise` with `error`.

[Using error scopes to capture validation errors from a](https://gpuweb.github.io#example-9d93bf47)

[GPUDevice](https://gpuweb.github.io#gpudevice)
gpuDevice. pushErrorScope( 'validation' ); let sampler= gpuDevice. createSampler({ maxAnisotropy: 0 , // Invalid, maxAnisotropy must be at least 1. }); gpuDevice. popErrorScope(). then(( error) => { if ( error) { // There was an error creating the sampler, so discard it. sampler= null ; console. error( \`An error occured while creating sampler: ${ error. message} \` ); } }); 

For example: An error scope that only contains the creation of a single resource, such as a texture or buffer, can be used to detect failures such as out of memory conditions, in which case the application may try freeing some resources and trying the allocation again.

Error scopes do not identify which command failed, however. So, for instance, wrapping all the commands executed while loading a model in a single error scope will not offer enough granularity to determine if the issue was due to memory constraints. As a result freeing resources would usually not be a productive response to a failure of that scope. A more appropriate response would be to allow the application to fall back to a different model or produce a warning that the model could not be loaded. If responding to memory constraints is desired, the operations allocating memory can always be wrapped in a smaller nested error scope.

### 22.4. Telemetry

When a [GPUError](https://gpuweb.github.io#gpuerror)[GPU error scope](https://gpuweb.github.io#gpu-error-scope), the user agent **may** [fire an event](https://dom.spec.whatwg.org/#concept-event-fire) named `uncapturederror` at a [GPUDevice](https://gpuweb.github.io#gpudevice)[GPUUncapturedErrorEvent](https://gpuweb.github.io#gpuuncapturederrorevent)

Note: [uncapturederror](https://gpuweb.github.io#eventdef-gpudevice-uncapturederror)[pushErrorScope()](https://gpuweb.github.io#dom-gpudevice-pusherrorscope)[popErrorScope()](https://gpuweb.github.io#dom-gpudevice-poperrorscope)

\[Exposed Window ,Worker ),SecureContext interface GPUUncapturedErrorEvent Event constructor DOMString type GPUUncapturedErrorEventInit gpuUncapturedErrorEventInitDict SameObject readonly attribute GPUError error dictionary GPUUncapturedErrorEventInit EventInit required GPUError error 

[GPUUncapturedErrorEvent](https://gpuweb.github.io#gpuuncapturederrorevent)

- `error`,  of type [GPUError](https://gpuweb.github.io#gpuerror), readonly
- A [slot-backed attribute](https://gpuweb.github.io#slot-backed-attribute) holding an object representing the error that was uncaptured.
This has the same type as errors returned by [popErrorScope()](https://gpuweb.github.io#dom-gpudevice-poperrorscope)

partial interface GPUDevice attribute EventHandler onuncapturederror 

[GPUDevice](https://gpuweb.github.io#gpudevice)[content timeline properties](https://gpuweb.github.io#content-timeline-property):

- `onuncapturederror`,  of type [EventHandler](https://html.spec.whatwg.org/multipage/webappapis.html#eventhandler)
- An [event handler IDL attribute](https://html.spec.whatwg.org/multipage/webappapis.html#event-handler-idl-attributes) for the [uncapturederror](https://gpuweb.github.io#eventdef-gpudevice-uncapturederror)

[Listening for uncaptured errors from a](https://gpuweb.github.io#example-bf3af559)

[GPUDevice](https://gpuweb.github.io#gpudevice)
gpuDevice. addEventListener( 'uncapturederror' , ( event) => { // Re-surface the error, because adding an event listener may silence console logs. console. error( 'A WebGPU error was not captured:' , event. error); myEngineDebugReport. uncapturedErrors. push({ type: event. error. constructor . name, message: event. error. message, }); }); 

## 23. Detailed Operations

This section describes the details of various GPU operations.

### 23.1. Computing

Computing operations provide direct access to GPU’s programmable hardware.
Compute shaders do not have shader stage inputs or outputs; their results are
side effects from writing data into storage bindings bound either as
[GPUBufferBindingLayout](https://gpuweb.github.io#dictdef-gpubufferbindinglayout)[GPUBufferBindingType](https://gpuweb.github.io#enumdef-gpubufferbindingtype)["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)[GPUStorageTextureBindingLayout](https://gpuweb.github.io#dictdef-gpustoragetexturebindinglayout)[GPUComputePassEncoder](https://gpuweb.github.io#gpucomputepassencoder)

The main compute algorithm:

`descriptor`,

`dispatchCall`)

**Arguments:**

- `descriptor`: Description of the current [GPUComputePipeline](https://gpuweb.github.io#gpucomputepipeline)
- `dispatchCall`: The dispatch call parameters. May come from function arguments or an [INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)

1. Let `computeInvocations` be an [empty](https://infra.spec.whatwg.org/#list-empty) [list](https://infra.spec.whatwg.org/#list).
2. Let `computeStage` be `descriptor`.[compute](https://gpuweb.github.io#dom-gpucomputepipelinedescriptor-compute)
3. Let `workgroupSize` be the computed workgroup size for `computeStage`.[entryPoint](https://gpuweb.github.io#dom-gpuprogrammablestage-entrypoint)`computeStage`.[constants](https://gpuweb.github.io#dom-gpuprogrammablestage-constants)`computeStage`.[module](https://gpuweb.github.io#dom-gpuprogrammablestage-module)
4. For `workgroupX` in range `[0,` :`dispatchCall`.`workgroupCountX`\]
  1. For `workgroupY` in range `[0,` :`dispatchCall`.`workgroupCountY`\]
    1. For `workgroupZ` in range `[0,` :`dispatchCall`.`workgroupCountZ`\]
      1. For `localX` in range `[0,` :`workgroupSize`.`x`\]
        1. For `localY` in range `[0,` :`workgroupSize`.`y`\]
          1. For `localZ` in range `[0,` :`workgroupSize`.`y`\]
            1. Let `invocation` be `{` `computeStage`, `workgroupX`, `workgroupY`, `workgroupZ`, `localX`, `localY`, `localZ` }
            2. [Append](https://infra.spec.whatwg.org/#list-append) `invocation` to `computeInvocations`.
5. For every `invocation` in `computeInvocations`, in any order the [device](https://gpuweb.github.io#device) chooses, including in parallel:
  1. Set the shader [builtins](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values):
    - Set the [num\_workgroups](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-num_workgroups) builtin, if any, to `(`
`dispatchCall`.`workgroupCountX`,
`dispatchCall`.`workgroupCountY`,
`dispatchCall`.`workgroupCountZ`
)
    - Set the [workgroup\_id](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-workgroup_id) builtin, if any, to `(`
`invocation`.`workgroupX`,
`invocation`.`workgroupY`,
`invocation`.`workgroupZ`
)
    - Set the [local\_invocation\_id](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-local_invocation_id) builtin, if any, to `(`
`invocation`.`localX`,
`invocation`.`localY`,
`invocation`.`localZ`
)
    - Set the [global\_invocation\_id](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-global_invocation_id) builtin, if any, to `(`.
`invocation`.`workgroupX` \* `workgroupSize`.`x` + `invocation`.`localX`,
`invocation`.`workgroupY` \* `workgroupSize`.`y` + `invocation`.`localY`,
`invocation`.`workgroupZ` \* `workgroupSize`.`z` + `invocation`.`localZ`
)
    - Set the [local\_invocation\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-local_invocation_index) builtin, if any, to ```
```
`invocation`.`localX` + (`invocation`.`localY` \* `workgroupSize`.`x`) +
(`invocation`.`localZ` \* `workgroupSize`.`x` \* `workgroupSize`.`y`)
  2. Invoke the compute shader entry point described by `invocation`.`computeStage`.

Note: Shader invocations have no guaranteed order, and will generally run in parallel according to device
    capabilities. Developers should not assume that any given invocation or workgroup will complete before any
    other one is started. Some devices may appear to execute in a consistent order, but this behavior should not
    be relied on as it will not perform identically across all devices. Shaders that require synchronization
    across invocations must use [Synchronization Built-in Functions](https://gpuweb.github.io/gpuweb/wgsl/#sync-builtin-functions) to coordinate execution.

The [device](https://gpuweb.github.io#device) may become [lost](https://gpuweb.github.io#lose-the-device) if
[shader execution does not end](https://gpuweb.github.io/gpuweb/wgsl/#shader-execution-end)
in a reasonable amount of time, as determined by the user agent.

### 23.2. Rendering

Rendering is done by a set of GPU operations that are executed within [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)

Note: rendering is the traditional use of GPUs, and is supported by multiple fixed-function blocks in hardware.

The main rendering algorithm:

**Arguments:**

- `pipeline`: The current [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
- `drawCall`: The draw call parameters. May come from function arguments or an [INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)
- `state`: [RenderState](https://gpuweb.github.io#renderstate) of the [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)

1. Let `descriptor` be `pipeline`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)
2. **Resolve indices**. See [§ 23.2.1 Index Resolution](https://gpuweb.github.io#index-resolution).
Let `vertexList` be the result of [resolve indices](https://gpuweb.github.io#abstract-opdef-resolve-indices)(`drawCall`, `state`).
3. **Process vertices**. See [§ 23.2.2 Vertex Processing](https://gpuweb.github.io#vertex-processing).
Execute [process vertices](https://gpuweb.github.io#abstract-opdef-process-vertices)(`vertexList`, `drawCall`, `descriptor`.[vertex](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-vertex)`state`).
4. **Assemble primitives**. See [§ 23.2.3 Primitive Assembly](https://gpuweb.github.io#primitive-assembly).
Execute [assemble primitives](https://gpuweb.github.io#abstract-opdef-assemble-primitives)(`vertexList`, `drawCall`, `descriptor`.[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)
5. **Clip primitives**. See [§ 23.2.4 Primitive Clipping](https://gpuweb.github.io#primitive-clipping).
Let `primitiveList` be the result of this stage.
6. **Rasterize**. See [§ 23.2.5 Rasterization](https://gpuweb.github.io#rasterization).
Let `rasterizationList` be the result of [rasterize](https://gpuweb.github.io#abstract-opdef-rasterize)(`primitiveList`, `state`).
7. **Process fragments**. See [§ 23.2.6 Fragment Processing](https://gpuweb.github.io#fragment-processing).
Gather a list of `fragments`, resulting from executing [process fragment](https://gpuweb.github.io#abstract-opdef-process-fragment)(`rasterPoint`, `descriptor`, `state`)
for each `rasterPoint` in `rasterizationList`.
8. **Write pixels**. See [§ 23.2.7 Output Merging](https://gpuweb.github.io#output-merging).
For each non-null `fragment` of `fragments`:
  - Execute [process depth stencil](https://gpuweb.github.io#abstract-opdef-process-depth-stencil)(`fragment`, `pipeline`, `state`).
  - Execute [process color attachments](https://gpuweb.github.io#abstract-opdef-process-color-attachments)(`fragment`, `pipeline`, `state`).

#### 23.2.1. Index Resolution

At the first stage of rendering, the pipeline builds a list of vertices to process for each instance.

**Arguments:**

- `drawCall`: The draw call parameters. May come from function arguments or an [INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)
- `state`: The snapshot of the [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)

**Returns:** list of integer indices.

1. Let `vertexIndexList` be an empty list of indices.
2. If `drawCall` is an indexed draw call:
  1. Initialize the `vertexIndexList` with `drawCall`.indexCount integers.
  2. For `i` in range 0 .. `drawCall`.indexCount (non-inclusive):
    1. Let `relativeVertexIndex` be [fetch index](https://gpuweb.github.io#abstract-opdef-fetch-index)(`i` + `drawCall`.`firstIndex`, `state`.[\[\[index\_buffer\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer-slot)
    2. If `relativeVertexIndex` has the special value `"out of bounds"`,
return the empty list.
Note: Implementations may choose to display a warning when this occurs, especially when it is easy to detect (like in non-indirect indexed draw calls).
    3. Append `drawCall`.`baseVertex` + `relativeVertexIndex` to the `vertexIndexList`.
Otherwise: 
  1. Initialize the `vertexIndexList` with `drawCall`.vertexCount integers.
  2. Set each `vertexIndexList` item `i` to the value `drawCall`.firstVertex + `i`.
3. Return `vertexIndexList`.

Note: in the case of indirect draw calls, the `indexCount`, `vertexCount`,
    and other properties of `drawCall` are read from the indirect buffer
    instead of the draw command itself.

**Arguments:**

- `i`: Index of a vertex index to fetch.
- `state`: The snapshot of the [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)

**Returns:** unsigned integer or `"out of bounds"`

1. Let `indexSize` be defined by the `state`.[\[\[index\_format\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_format-slot)
2. If `state`.[\[\[index\_buffer\_offset\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer_offset-slot)`indexSize` > `state`.[\[\[index\_buffer\_size\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer_size-slot)`"out of bounds"`.
3. Interpret the data in `state`.[\[\[index\_buffer\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer-slot)`state`.[\[\[index\_buffer\_offset\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-index_buffer_offset-slot)`i` × `indexSize`,
of size `indexSize` bytes, as an unsigned integer and return it.

#### 23.2.2. Vertex Processing

Vertex processing stage is a programmable stage of the render [pipeline](https://gpuweb.github.io#pipeline) that
processes the vertex attribute data, and produces
clip space positions for [§ 23.2.4 Primitive Clipping](https://gpuweb.github.io#primitive-clipping), as well as other data for the
[§ 23.2.6 Fragment Processing](https://gpuweb.github.io#fragment-processing).

**Arguments:**

- `vertexIndexList`: List of vertex indices to process (mutable, passed by reference).
- `drawCall`: The draw call parameters. May come from function arguments or an [INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)
- `desc`: The descriptor of type [GPUVertexState](https://gpuweb.github.io#dictdef-gpuvertexstate)
- `state`: The snapshot of the [GPURenderCommandsMixin](https://gpuweb.github.io#gpurendercommandsmixin)

Each vertex `vertexIndex` in the `vertexIndexList`,
    in each instance of index `rawInstanceIndex`, is processed independently.
    The `rawInstanceIndex` is in range from 0 to `drawCall`.instanceCount - 1, inclusive.
    This processing happens in parallel, and any side effects, such as
    writes into [GPUBufferBindingType](https://gpuweb.github.io#enumdef-gpubufferbindingtype)["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)

1. Let `instanceIndex` be `rawInstanceIndex` + `drawCall`.firstInstance.
2. For each non-`null` `vertexBufferLayout` in the list of `desc`.[buffers](https://gpuweb.github.io#dom-gpuvertexstate-buffers)
  1. Let `i` be the index of the buffer layout in this list.
  2. Let `vertexBuffer`, `vertexBufferOffset`, and `vertexBufferBindingSize` be the
buffer, offset, and size at slot `i` of `state`.[\[\[vertex\_buffers\]\]](https://gpuweb.github.io#dom-gpurendercommandsmixin-vertex_buffers-slot)
  3. Let `vertexElementIndex` be dependent on `vertexBufferLayout`.[stepMode](https://gpuweb.github.io#dom-gpuvertexbufferlayout-stepmode)
    - ["vertex"](https://gpuweb.github.io#dom-gpuvertexstepmode-vertex)
    - `vertexIndex`
    - ["instance"](https://gpuweb.github.io#dom-gpuvertexstepmode-instance)
    - `instanceIndex`
  4. Let `drawCallOutOfBounds` be `false`.
  5. For each `attributeDesc` in `vertexBufferLayout`.[attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes)
    1. Let `attributeOffset` be `vertexBufferOffset` + `vertexElementIndex` \* `vertexBufferLayout`.[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)`attributeDesc`.[offset](https://gpuweb.github.io#dom-gpuvertexattribute-offset)
    2. If `attributeOffset` + [byteSize](https://gpuweb.github.io#abstract-opdef-gpuvertexformat-bytesize)(`attributeDesc`.[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)`vertexBufferOffset` + `vertexBufferBindingSize`:
      1. Set `drawCallOutOfBounds` to `true`.
      2. **Optionally ([implementation-defined](https://infra.spec.whatwg.org/#implementation-defined))**, [empty](https://infra.spec.whatwg.org/#list-empty) `vertexIndexList` and return, cancelling the draw call.
Note: This allows implementations to detect out-of-bounds values in the index buffer
before issuing a draw call, instead of using [invalid memory reference](https://gpuweb.github.io/gpuweb/wgsl/#invalid-memory-reference) behavior.
  6. For each `attributeDesc` in `vertexBufferLayout`.[attributes](https://gpuweb.github.io#dom-gpuvertexbufferlayout-attributes)
    1. If `drawCallOutOfBounds` is `true`:
      1. Load the attribute `data` according to WGSL’s [invalid memory reference](https://gpuweb.github.io/gpuweb/wgsl/#invalid-memory-reference) behavior, from `vertexBuffer`.
Note: [Invalid memory reference](https://gpuweb.github.io/gpuweb/wgsl/#invalid-memory-reference) allows several behaviors, including actually
loading the "correct" result for an attribute that is in-bounds, even when
the draw-call-wide `drawCallOutOfBounds` is `true`.
Otherwise: 
      1. Let `attributeOffset` be `vertexBufferOffset` + `vertexElementIndex` \* `vertexBufferLayout`.[arrayStride](https://gpuweb.github.io#dom-gpuvertexbufferlayout-arraystride)`attributeDesc`.[offset](https://gpuweb.github.io#dom-gpuvertexattribute-offset)
      2. Load the attribute `data` of format `attributeDesc`.[format](https://gpuweb.github.io#dom-gpuvertexattribute-format)`vertexBuffer` starting at offset `attributeOffset`.
The components are loaded in the order `x`, `y`, `z`, `w` from buffer memory.
    2. Convert the `data` into a shader-visible format, according to [channel formats](https://gpuweb.github.io/gpuweb/wgsl/#channel-formats) rules.
["snorm8x2"](https://gpuweb.github.io#dom-gpuvertexformat-snorm8x2)
`[0x70, 0xD0]` will be converted to `vec2<f32>(0.88, -0.38)` in WGSL.
    3. Adjust the `data` size to the shader type:
      - if both are scalar, or both are vectors of the same dimensionality, no adjustment is needed.
      - if `data` is vector but the shader type is scalar, then only the first component is extracted.
      - if both are vectors, and `data` has a higher dimension, the extra components are dropped.
["float32x3"](https://gpuweb.github.io#dom-gpuvertexformat-float32x3)
`vec3<f32>(1.0, 2.0, 3.0)` will exposed to the shader as `vec2<f32>(1.0, 2.0)` if a 2-component vector is expected.
      - if the shader type is a vector of higher dimensionality, or the `data` is a scalar,
then the missing components are filled from `vec4<*>(0, 0, 0, 1)` value.
["sint32"](https://gpuweb.github.io#dom-gpuvertexformat-sint32)
`5` will be exposed
    to the shader as `vec4<i32>(5, 0, 0, 1)` if a 4-component vector is expected.
    4. Bind the `data` to vertex shader input
location `attributeDesc`.[shaderLocation](https://gpuweb.github.io#dom-gpuvertexattribute-shaderlocation)
3. For each [GPUBindGroup](https://gpuweb.github.io#gpubindgroup)`index` in `state`.[\[\[bind\_groups\]\]](https://gpuweb.github.io#dom-gpubindingcommandsmixin-bind_groups-slot)
  1. For each resource [GPUBindingResource](https://gpuweb.github.io#typedefdef-gpubindingresource)
    1. Let `entry` be the corresponding [GPUBindGroupLayoutEntry](https://gpuweb.github.io#dictdef-gpubindgrouplayoutentry)
    2. If `entry`.[visibility](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-visibility)[VERTEX](https://gpuweb.github.io#dom-gpushaderstage-vertex)
      - Bind the resource to the shader under group `index` and binding [GPUBindGroupLayoutEntry.binding](https://gpuweb.github.io#dom-gpubindgrouplayoutentry-binding)
4. Set the shader [builtins](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values):
  - Set the `vertex_index` builtin, if any, to `vertexIndex`.
  - Set the `instance_index` builtin, if any, to `instanceIndex`.
5. Invoke the vertex shader entry point described by `desc`.
Note: The target platform caches the results of vertex shader invocations.
There is no guarantee that any `vertexIndex` that repeats more than once will
result in multiple invocations. Similarly, there is no guarantee that a single `vertexIndex` will only be processed once.
The [device](https://gpuweb.github.io#device) may become [lost](https://gpuweb.github.io#lose-the-device) if [shader execution does not end](https://gpuweb.github.io/gpuweb/wgsl/#shader-execution-end) in a reasonable amount of time, as determined by the user agent.

#### 23.2.3. Primitive Assembly

Primitives are assembled by a fixed-function stage of GPUs.

**Arguments:**

- `vertexIndexList`: List of vertex indices to process.
- `drawCall`: The draw call parameters. May come from function arguments or an [INDIRECT](https://gpuweb.github.io#dom-gpubufferusage-indirect)
- `desc`: The descriptor of type [GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate)

For each instance, the primitives get assembled from the vertices that have been
    processed by the shaders, based on the `vertexIndexList`.

1. First, if the primitive topology is a strip, (which means that
`desc`.[stripIndexFormat](https://gpuweb.github.io#dom-gpuprimitivestate-stripindexformat)`drawCall` is indexed, the `vertexIndexList` is split into
sub-lists using the maximum value of `desc`.[stripIndexFormat](https://gpuweb.github.io#dom-gpuprimitivestate-stripindexformat)
Example: a `vertexIndexList` with values `[1, 2, 65535, 4, 5, 6]` of type ["uint16"](https://gpuweb.github.io#dom-gpuindexformat-uint16)`[1, 2]` and `[4, 5, 6]`.
2. For each of the sub-lists `vl`, primitive generation is done according to the `desc`.[topology](https://gpuweb.github.io#dom-gpuprimitivestate-topology)
  - ["line-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-list)
  - Line primitives are composed from (`vl`.0, `vl`.1),
then (`vl`.2, `vl`.3), then (`vl`.4 to `vl`.5), etc.
Each subsequent primitive takes 2 vertices.
  - ["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)
  - Line primitives are composed from (`vl`.0, `vl`.1),
then (`vl`.1, `vl`.2), then (`vl`.2, `vl`.3), etc.
Each subsequent primitive takes 1 vertex.
  - ["triangle-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-list)
  - Triangle primitives are composed from (`vl`.0, `vl`.1, `vl`.2),
then (`vl`.3, `vl`.4, `vl`.5), then (`vl`.6, `vl`.7, `vl`.8), etc.
Each subsequent primitive takes 3 vertices.
  - ["triangle-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-strip)
  - Triangle primitives are composed from (`vl`.0, `vl`.1, `vl`.2),
then (`vl`.2, `vl`.1, `vl`.3), then (`vl`.2, `vl`.3, `vl`.4),
then (`vl`.4, `vl`.3, `vl`.5), etc.
Each subsequent primitive takes 1 vertices.
Any incomplete primitives are dropped.

#### 23.2.4. Primitive Clipping

Vertex shaders have to produce a built-in [position](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-position) (of type `vec4<f32>`),
which denotes the clip position of a vertex in [clip space coordinates](https://gpuweb.github.io#clip-space-coordinates).

Primitives are clipped to the clip volume, which, for any [clip position](https://gpuweb.github.io#clip-position) `p`
inside a primitive, is defined by the following inequalities:

- −`p`.w ≤ `p`.x ≤ `p`.w
- −`p`.w ≤ `p`.y ≤ `p`.w
- 0 ≤ `p`.z ≤ `p`.w (depth clipping)

When the ["clip-distances"](https://gpuweb.github.io#dom-gpufeaturename-clip-distances)[clip volume](https://gpuweb.github.io#clip-volume) can
be further restricted by user-defined half-spaces by declaring [clip\_distances](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-clip_distances) in the
output of vertex stage. Each value in the [clip\_distances](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-clip_distances) array will be linearly
interpolated across the primitive, and the portion of the primitive with interpolated distances less
than 0 will be clipped.

If `descriptor`.[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[unclippedDepth](https://gpuweb.github.io#dom-gpuprimitivestate-unclippeddepth)`true`,
[depth clipping](https://gpuweb.github.io#depth-clipping) is not applied: the [clip volume](https://gpuweb.github.io#clip-volume) is not bounded in the z dimension.

A primitive passes through this stage unchanged if every one of its edges
lie entirely inside the [clip volume](https://gpuweb.github.io#clip-volume).
If the edges of a primitives intersect the boundary of the [clip volume](https://gpuweb.github.io#clip-volume),
the intersecting edges are reconnected by new edges that lie along the boundary of the [clip volume](https://gpuweb.github.io#clip-volume).
For triangular primitives (`descriptor`.[primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[topology](https://gpuweb.github.io#dom-gpuprimitivestate-topology)["triangle-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-list)["triangle-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-strip)

If a primitive intersects an edge of the [clip volume](https://gpuweb.github.io#clip-volume)’s boundary,
the clipped polygon must include a point on this boundary edge.

If the vertex shader outputs other floating-point values (scalars and vectors), qualified with "perspective" interpolation, they also get clipped. The output values associated with a vertex that lies within the clip volume are unaffected by clipping. If a primitive is clipped, however, the output values assigned to vertices produced by clipping are clipped.

Considering an edge between vertices `a` and `b` that got clipped, resulting in the vertex `c`,
let’s define `t` to be the ratio between the edge vertices:
`c`.p = `t` × `a`.p + (1 − `t`) × `b`.p,
where `x`.p is the output [clip position](https://gpuweb.github.io#clip-position) of a vertex `x`.

For each vertex output value "v" with a corresponding fragment input,
`a`.v and `b`.v would be the outputs for `a` and `b` vertices respectively.
The clipped shader output `c`.v is produced based on the interpolation qualifier:

- [flat](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-type-flat)
- Flat interpolation is unaffected, and is based on the provoking vertex,
which is determined by the [interpolation sampling](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-sampling) mode declared in the shader. The
output value is the same for the whole primitive, and matches the vertex output of the [provoking vertex](https://gpuweb.github.io#provoking-vertex).
- [linear](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-type-linear)
- The interpolation ratio gets adjusted against the perspective coordinates of the
[clip position](https://gpuweb.github.io#clip-position)s, so that the result of interpolation is linear in screen space.
- [perspective](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-type-perspective)
- The value is linearly interpolated in clip space, producing perspective-correct values.

The result of primitive clipping is a new set of primitives, which are contained
within the [clip volume](https://gpuweb.github.io#clip-volume).

#### 23.2.5. Rasterization

Rasterization is the hardware processing stage that maps the generated primitives
to the 2-dimensional rendering area of the framebuffer -
the set of render attachments in the current [GPURenderPassEncoder](https://gpuweb.github.io#gpurenderpassencoder)

The [framebuffer](https://gpuweb.github.io#framebuffer) coordinates start from the top-left corner of the render targets.
Each unit corresponds exactly to one pixel. See [§ 3.3 Coordinate Systems](https://gpuweb.github.io#coordinate-systems) for more information.

Rasterization determines the set of pixels affected by a primitive. In case of multi-sampling,
each pixel is further split into
`descriptor`.[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)[count](https://gpuweb.github.io#dom-gpumultisamplestate-count)

| [multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample).[count](https://gpuweb.github.io#dom-gpumultisamplestate-count) | Sample positions | 
|---|---|
| 1 | Sample 0: (0.5, 0.5) | 
| 4 | Sample 0: (0.375, 0.125) Sample 1: (0.875, 0.375) Sample 2: (0.125, 0.625) Sample 3: (0.625, 0.875) | 

Implementations must use the [standard sample pattern](https://gpuweb.github.io#standard-sample-patterns) for the given
[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)[count](https://gpuweb.github.io#dom-gpumultisamplestate-count)

Let’s define a FragmentDestination to contain:

- position
- the 2D pixel position using [framebuffer coordinates](https://gpuweb.github.io#framebuffer-coordinates)
- sampleIndex
- an integer in case [§ 23.2.10 Per-Sample Shading](https://gpuweb.github.io#per-sample-shading) is active,
or `null` otherwise

We’ll also use a notion of [normalized device coordinates](https://gpuweb.github.io#ndc), or NDC.
In this coordinate system, the viewport bounds range in X and Y from -1 to 1, and in Z from 0 to 1.

Rasterization produces a list of RasterizationPoints, each containing the following data:

- destination
- refers to [FragmentDestination](https://gpuweb.github.io#fragmentdestination)
- coverageMask
- refers to multisample coverage mask (see [§ 23.2.11 Sample Masking](https://gpuweb.github.io#sample-masking))
- frontFacing
- is true if it’s a point on the front face of a primitive
- perspectiveDivisor
- refers to interpolated 1.0 ÷ W across the primitive
- depth
- refers to the depth in [viewport coordinates](https://gpuweb.github.io#viewport-coordinates),
i.e. between the [\[\[viewport\]\]](https://gpuweb.github.io#dom-renderstate-viewport-slot)`minDepth` and `maxDepth`.
- primitiveVertices
- refers to the list of vertex outputs forming the primitive
- barycentricCoordinates
- refers to [§ 23.2.5.3 Barycentric coordinates](https://gpuweb.github.io#barycentric-coordinates)

**Arguments:**

- `primitiveList`: List of primitives to rasterize.
- `state`: The active [RenderState](https://gpuweb.github.io#renderstate).

**Returns:** list of [RasterizationPoint](https://gpuweb.github.io#rasterizationpoint).

Each primitive in `primitiveList` is processed independently.
    However, the order of primitives affects later stages, such as depth/stencil operations and pixel writes.

1. First, the clipped vertices are transformed into [NDC](https://gpuweb.github.io#ndc) - normalized device coordinates.
Given the output position `p`, the [NDC](https://gpuweb.github.io#ndc) position and perspective divisor are:
ndc(`p`) = vector(`p`.x ÷ `p`.w, `p`.y ÷ `p`.w, `p`.z ÷ `p`.w)
divisor(`p`) = 1.0 ÷ `p`.w
2. Let `vp` be `state`.[\[\[viewport\]\]](https://gpuweb.github.io#dom-renderstate-viewport-slot)[NDC](https://gpuweb.github.io#ndc) position `n` into [viewport coordinates](https://gpuweb.github.io#viewport-coordinates):
  - Compute [framebuffer](https://gpuweb.github.io#framebuffer) coordinates from the render target offset and size:
framebufferCoords(`n`) = vector(`vp`.`x` + 0.5 × (`n`.x + 1) × `vp`.`width`, `vp`.`y` + 0.5 × (−`n`.y + 1) × `vp`.`height`)
  - Compute depth by linearly mapping \[0,1\] to the viewport depth range: depth(`n`) = `vp`.`minDepth` + `n`.`z` × ( `vp`.`maxDepth` - `vp`.`minDepth` )
3. Let `rasterizationPoints` be the list of points, each having its attributes (`divisor(p)`, `framebufferCoords(n)`, `depth(n)`, etc.) interpolated according to its position on the
primitive, using the same interpolation as [§ 23.2.4 Primitive Clipping](https://gpuweb.github.io#primitive-clipping). If the attribute is
user-defined (not a [built-in output value](https://gpuweb.github.io/gpuweb/wgsl/#built-in-output-value)) then the [interpolation type](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-type) specified by
the [@interpolate](https://gpuweb.github.io/gpuweb/wgsl/#interpolate-attr) WGSL attribute is used.
4. Proceed with a specific rasterization algorithm,
depending on [primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[topology](https://gpuweb.github.io#dom-gpuprimitivestate-topology)
  - ["point-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-point-list)
  - The point, if not filtered by [§ 23.2.4 Primitive Clipping](https://gpuweb.github.io#primitive-clipping), goes into [§ 23.2.5.1 Point Rasterization](https://gpuweb.github.io#point-rasterization).
  - ["line-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-list)
["line-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-line-strip)
  - The line cut by [§ 23.2.4 Primitive Clipping](https://gpuweb.github.io#primitive-clipping) goes into [§ 23.2.5.2 Line Rasterization](https://gpuweb.github.io#line-rasterization).
  - ["triangle-list"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-list)
["triangle-strip"](https://gpuweb.github.io#dom-gpuprimitivetopology-triangle-strip)
  - The polygon produced in [§ 23.2.4 Primitive Clipping](https://gpuweb.github.io#primitive-clipping) goes into [§ 23.2.5.4 Polygon Rasterization](https://gpuweb.github.io#polygon-rasterization).
5. Remove all the points `rp` from `rasterizationPoints` that have `rp`.[destination](https://gpuweb.github.io#rasterizationpoint-destination).[position](https://gpuweb.github.io#fragmentdestination-position) outside of `state`.[\[\[scissorRect\]\]](https://gpuweb.github.io#dom-renderstate-scissorrect-slot)
6. Return `rasterizationPoints`.

##### 23.2.5.1. Point Rasterization

A single [FragmentDestination](https://gpuweb.github.io#fragmentdestination) is selected within the pixel containing the
[framebuffer](https://gpuweb.github.io#framebuffer) coordinates of the point.

The coverage mask depends on multi-sampling mode:

- sample-frequency
- coverageMask = 1 ≪ `sampleIndex`
- pixel-frequency multi-sampling
- coverageMask = 1 ≪ `descriptor`.[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)[count](https://gpuweb.github.io#dom-gpumultisamplestate-count)
- no multi-sampling
- coverageMask = 1

##### 23.2.5.2. Line Rasterization

The exact algorithm used for line rasterization is not defined, and may differ between
implementations. For example, the line may be drawn using [§ 23.2.5.4 Polygon Rasterization](https://gpuweb.github.io#polygon-rasterization) of a 1px-width
rectangle around the line segment, or using Bresenham’s line algorithm to select the
[FragmentDestination](https://gpuweb.github.io#fragmentdestination)s.

Note: See [Basic Line Segment Rasterization](https://registry.khronos.org/vulkan/specs/1.3/html/vkspec.html#primsrast-lines-basic) and
[Bresenham Line Segment Rasterization](https://registry.khronos.org/vulkan/specs/1.3/html/vkspec.html#primsrast-lines-bresenham) in the [Vulkan 1.3](https://registry.khronos.org/vulkan/specs/1.3/html/vkspec.html)
spec for more details of how line these line rasterization algorithms may be implemented.

##### 23.2.5.3. Barycentric coordinates

Barycentric coordinates is a list of `n` numbers `b`<sub>`i`</sub>,
defined for a point `p` inside a convex polygon with `n` vertices `v`<sub>`i`</sub> in [framebuffer](https://gpuweb.github.io#framebuffer) space.
Each `b`<sub>`i`</sub> is in range 0 to 1, inclusive, and represents the proximity to vertex `v`<sub>`i`</sub>.
Their sum is always constant:

∑ (`b`<sub>`i`</sub>) = 1

These coordinates uniquely specify any point `p` within the polygon (or on its boundary) as:

`p` = ∑ (`b`<sub>`i`</sub> × `p`<sub>`i`</sub>)

For a polygon with 3 vertices - a triangle,
barycentric coordinates of any point `p` can be computed as follows:

`A`<sub>polygon</sub> = A(`v`<sub>`1`</sub>, `v`<sub>`2`</sub>, `v`<sub>`3`</sub>)
`b`<sub>`1`</sub> = A(`p`, `b`<sub>`2`</sub>, `b`<sub>`3`</sub>) ÷ `A`<sub>polygon</sub>
`b`<sub>`2`</sub> = A(`b`<sub>`1`</sub>, `p`, `b`<sub>`3`</sub>) ÷ `A`<sub>polygon</sub>
`b`<sub>`3`</sub> = A(`b`<sub>`1`</sub>, `b`<sub>`2`</sub>, `p`) ÷ `A`<sub>polygon</sub>

Where A(list of points) is the area of the polygon with the given set of vertices.

For polygons with more than 3 vertices, the exact algorithm is implementation-dependent. One of the possible implementations is to triangulate the polygon and compute the barycentrics of a point based on the triangle it falls into.

##### 23.2.5.4. Polygon Rasterization

A polygon is front-facing if it’s oriented towards the projection. Otherwise, the polygon is back-facing.

**Arguments:**

**Returns:** list of [RasterizationPoint](https://gpuweb.github.io#rasterizationpoint).

1. Let `rasterizationPoints` be an empty list.
2. Let `v`(`i`) be the [framebuffer](https://gpuweb.github.io#framebuffer) coordinates for the clipped vertex number `i` (starting with 1)
in a rasterized polygon of `n` vertices.
Note: this section uses the term "polygon" instead of a "triangle",
since [§ 23.2.4 Primitive Clipping](https://gpuweb.github.io#primitive-clipping) stage may have introduced additional vertices.
This is non-observable by the application.
3. Determine if the polygon is front-facing,
which depends on the sign of the `area` occupied by the polygon in [framebuffer](https://gpuweb.github.io#framebuffer) coordinates:
`area` = 0.5 × ((`v`<sub>1</sub>.x × `v`<sub>`n`</sub>.y − `v`<sub>`n`</sub>.x × `v`<sub>1</sub>.y) + ∑ (`v`<sub>`i`+1</sub>.x × `v`<sub>`i`</sub>.y − `v`<sub>`i`</sub>.x × `v`<sub>`i`+1</sub>.y))
The sign of `area` is interpreted based on the [primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[frontFace](https://gpuweb.github.io#dom-gpuprimitivestate-frontface)
  - ["ccw"](https://gpuweb.github.io#dom-gpufrontface-ccw)
  - `area` > 0 is considered [front-facing](https://gpuweb.github.io#front-facing), otherwise [back-facing](https://gpuweb.github.io#back-facing)
  - ["cw"](https://gpuweb.github.io#dom-gpufrontface-cw)
  - `area` \< 0 is considered [front-facing](https://gpuweb.github.io#front-facing), otherwise [back-facing](https://gpuweb.github.io#back-facing)
4. Cull based on [primitive](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-primitive)[cullMode](https://gpuweb.github.io#dom-gpuprimitivestate-cullmode)
  - ["none"](https://gpuweb.github.io#dom-gpucullmode-none)
  - All polygons pass this test.
  - ["front"](https://gpuweb.github.io#dom-gpucullmode-front)
  - The [front-facing](https://gpuweb.github.io#front-facing) polygons are discarded,
and do not process in later stages of the render pipeline.
  - ["back"](https://gpuweb.github.io#dom-gpucullmode-back)
  - The [back-facing](https://gpuweb.github.io#back-facing) polygons are discarded.
5. Determine a set of [fragments](https://gpuweb.github.io#fragment) inside the polygon in [framebuffer](https://gpuweb.github.io#framebuffer) space -
these are locations scheduled for the per-fragment operations.
This operation is known as "point sampling".
The logic is based on `descriptor`.[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)
  - disabled
  - [Fragment](https://gpuweb.github.io#fragment)s are associated with pixel centers. That is, all the points with coordinates `C`, where
fract(`C`) = vector2(0.5, 0.5) in the [framebuffer](https://gpuweb.github.io#framebuffer) space, enclosed into the polygon, are included.
If a pixel center is on the edge of the polygon, whether or not it’s included is not defined.
Note: this becomes a subject of precision for the rasterizer.
  - enabled
  - Each pixel is associated with `descriptor`.[multisample](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-multisample)[count](https://gpuweb.github.io#dom-gpumultisamplestate-count)[implementation-defined](https://infra.spec.whatwg.org/#implementation-defined).
The locations are ordered, and the list is the same for each pixel of the [framebuffer](https://gpuweb.github.io#framebuffer).
Each location corresponds to one fragment in the multisampled [framebuffer](https://gpuweb.github.io#framebuffer).
The rasterizer builds a mask of locations being hit inside each pixel and provides is as "sample-mask" built-in to the fragment shader.
6. For each produced fragment of type [FragmentDestination](https://gpuweb.github.io#fragmentdestination):
  1. Let `rp` be a new [RasterizationPoint](https://gpuweb.github.io#rasterizationpoint) object
  2. Compute the list `b` as [§ 23.2.5.3 Barycentric coordinates](https://gpuweb.github.io#barycentric-coordinates) of that fragment.
Set `rp`.[barycentricCoordinates](https://gpuweb.github.io#rasterizationpoint-barycentriccoordinates) to `b`.
  3. Let `d`<sub>`i`</sub> be the depth value of `v`<sub>`i`</sub>.
  4. Set `rp`.[depth](https://gpuweb.github.io#rasterizationpoint-depth) to ∑ (`b`<sub>`i`</sub> × `d`<sub>`i`</sub>)
  5. Append `rp` to `rasterizationPoints`.
7. Return `rasterizationPoints`.

#### 23.2.6. Fragment Processing

The fragment processing stage is a programmable stage of the render [pipeline](https://gpuweb.github.io#pipeline) that
computes the fragment data (often a color) to be written into render targets.

This stage produces a Fragment for each [RasterizationPoint](https://gpuweb.github.io#rasterizationpoint):

- destination refers to [FragmentDestination](https://gpuweb.github.io#fragmentdestination).
- frontFacing is true if it’s a fragment on the front face of a primitive.
- coverageMask refers to multisample coverage mask (see [§ 23.2.11 Sample Masking](https://gpuweb.github.io#sample-masking)).
- depth refers to the depth in [viewport coordinates](https://gpuweb.github.io#viewport-coordinates),
i.e. between the [\[\[viewport\]\]](https://gpuweb.github.io#dom-renderstate-viewport-slot)`minDepth` and `maxDepth`.
- colors refers to the list of color values,
one for each target in [colorAttachments](https://gpuweb.github.io#dom-gpurenderpassdescriptor-colorattachments)
- depthPassed
is `true` if the fragment passed the [depthCompare](https://gpuweb.github.io#dom-gpudepthstencilstate-depthcompare)
- stencilPassed
is `true` if  the fragment passed the stencil [compare](https://gpuweb.github.io#dom-gpustencilfacestate-compare)

**Arguments:**

- `rp`: The [RasterizationPoint](https://gpuweb.github.io#rasterizationpoint), produced by [§ 23.2.5 Rasterization](https://gpuweb.github.io#rasterization).
- `descriptor`: The descriptor of type [GPURenderPipelineDescriptor](https://gpuweb.github.io#dictdef-gpurenderpipelinedescriptor)
- `state`: The active [RenderState](https://gpuweb.github.io#renderstate).

**Returns:** [Fragment](https://gpuweb.github.io#fragment) or `null`.

1. Let `fragmentDesc` be `descriptor`.[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)
2. Let `depthStencilDesc` be `descriptor`.[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)
3. Let `fragment` be a new [Fragment](https://gpuweb.github.io#fragment) object.
4. Set `fragment`.[destination](https://gpuweb.github.io#fragment-destination) to `rp`.[destination](https://gpuweb.github.io#rasterizationpoint-destination).
5. Set `fragment`.[frontFacing](https://gpuweb.github.io#fragment-frontfacing) to `rp`.[frontFacing](https://gpuweb.github.io#rasterizationpoint-frontfacing).
6. Set `fragment`.[coverageMask](https://gpuweb.github.io#fragment-coveragemask) to `rp`.[coverageMask](https://gpuweb.github.io#rasterizationpoint-coveragemask).
7. Set `fragment`.[depth](https://gpuweb.github.io#fragment-depth) to `rp`.[depth](https://gpuweb.github.io#rasterizationpoint-depth).
8. If `frag_depth` [builtin](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values) is not produced by the shader:
  1. Set `fragment`.[depthPassed](https://gpuweb.github.io#fragment-depthpassed) to the result of [compare fragment](https://gpuweb.github.io#abstract-opdef-compare-fragment)(`fragment`.[destination](https://gpuweb.github.io#fragment-destination), `fragment`.[depth](https://gpuweb.github.io#fragment-depth), "[depth](https://gpuweb.github.io#aspect-depth)", `state`.[\[\[depthStencilAttachment\]\]](https://gpuweb.github.io#dom-renderstate-depthstencilattachment-slot)`depthStencilDesc`?.[depthCompare](https://gpuweb.github.io#dom-gpudepthstencilstate-depthcompare)
9. Set `stencilState` to `depthStencilDesc`?.[stencilFront](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilfront)`rp`.[frontFacing](https://gpuweb.github.io#rasterizationpoint-frontfacing) is `true` and `depthStencilDesc`?.[stencilBack](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilback)
10. Set `fragment`.[stencilPassed](https://gpuweb.github.io#fragment-stencilpassed) to the result of [compare fragment](https://gpuweb.github.io#abstract-opdef-compare-fragment)(`fragment`.[destination](https://gpuweb.github.io#fragment-destination), `state`.[\[\[stencilReference\]\]](https://gpuweb.github.io#dom-renderstate-stencilreference-slot)[stencil](https://gpuweb.github.io#aspect-stencil)", `state`.[\[\[depthStencilAttachment\]\]](https://gpuweb.github.io#dom-renderstate-depthstencilattachment-slot)`stencilState`?.[compare](https://gpuweb.github.io#dom-gpustencilfacestate-compare)
11. If `fragmentDesc` is not `null`:
  1. If `fragment`.[depthPassed](https://gpuweb.github.io#fragment-depthpassed) is `false`, the `frag_depth` [builtin](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values) is not produced by the
shader entry point, and the shader entry point does not write to any [storage](https://gpuweb.github.io#internal-usage-storage) bindings,
the following steps may be skipped.
  2. Set the shader input [builtins](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values). For each non-composite argument of the entry point,
annotated as a [builtin](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values), set its value based on the annotation:
    - `position`
    - `vec4<f32>`(`rp`.[destination](https://gpuweb.github.io#rasterizationpoint-destination).[position](https://gpuweb.github.io#fragmentdestination-position), `rp`.[depth](https://gpuweb.github.io#rasterizationpoint-depth), `rp`.[perspectiveDivisor](https://gpuweb.github.io#rasterizationpoint-perspectivedivisor))
    - `front_facing`
    - `rp`.[frontFacing](https://gpuweb.github.io#rasterizationpoint-frontfacing)
    - `sample_index`
    - `rp`.[destination](https://gpuweb.github.io#rasterizationpoint-destination).[sampleIndex](https://gpuweb.github.io#fragmentdestination-sampleindex)
    - `sample_mask`
    - `rp`.[coverageMask](https://gpuweb.github.io#rasterizationpoint-coveragemask)
  3. For each user-specified [shader stage input](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-input) of the fragment stage:
    1. Let `value` be the interpolated fragment input,
based on `rp`.[barycentricCoordinates](https://gpuweb.github.io#rasterizationpoint-barycentriccoordinates), `rp`.[primitiveVertices](https://gpuweb.github.io#rasterizationpoint-primitivevertices),
and the [interpolation](https://gpuweb.github.io/gpuweb/wgsl/#interpolation) qualifier on the input.
    2. Set the corresponding fragment shader [location](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) input to `value`.
  4. Invoke the fragment shader entry point described by `fragmentDesc`.
The [device](https://gpuweb.github.io#device) may become [lost](https://gpuweb.github.io#lose-the-device) if [shader execution does not end](https://gpuweb.github.io/gpuweb/wgsl/#shader-execution-end) in a reasonable amount of time, as determined by the user agent.
  5. If the fragment issued `discard`, return `null`.
  6. Set `fragment`.[colors](https://gpuweb.github.io#fragment-colors) to the user-specified [shader stage output](https://gpuweb.github.io/gpuweb/wgsl/#shader-stage-output) values from the shader.
  7. Take the shader output [builtins](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values):
    1. If `frag_depth` [builtin](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values) is produced by the shader as `value`:
      1. Let `vp` be `state`.[\[\[viewport\]\]](https://gpuweb.github.io#dom-renderstate-viewport-slot)
      2. Set `fragment`.[depth](https://gpuweb.github.io#fragment-depth) to clamp(`value`, `vp`.`minDepth`, `vp`.`maxDepth`).
      3. Set `fragment`.[depthPassed](https://gpuweb.github.io#fragment-depthpassed) to the result of [compare fragment](https://gpuweb.github.io#abstract-opdef-compare-fragment)(`fragment`.[destination](https://gpuweb.github.io#fragment-destination), `fragment`.[depth](https://gpuweb.github.io#fragment-depth), "[depth](https://gpuweb.github.io#aspect-depth)", `state`.[\[\[depthStencilAttachment\]\]](https://gpuweb.github.io#dom-renderstate-depthstencilattachment-slot)`depthStencilDesc`?.[depthCompare](https://gpuweb.github.io#dom-gpudepthstencilstate-depthcompare)
  8. If `sample_mask` [builtin](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values) is produced by the shader as `value`:
    1. Set `fragment`.[coverageMask](https://gpuweb.github.io#fragment-coveragemask) to `fragment`.[coverageMask](https://gpuweb.github.io#fragment-coveragemask) ∧ `value`.
Otherwise we are in [§ 23.2.8 No Color Output](https://gpuweb.github.io#no-color-output) mode, and `fragment`.[colors](https://gpuweb.github.io#fragment-colors) is empty.
12. Return `fragment`.

**Arguments:**

- `destination`: The [FragmentDestination](https://gpuweb.github.io#fragmentdestination).
- `value`: The value to be compared.
- `aspect`: The [aspect](https://gpuweb.github.io#aspect) of `attachment` to sample values from.
- `attachment`: The attachment to be compared against.
- `compareFunc`: The [GPUCompareFunction](https://gpuweb.github.io#enumdef-gpucomparefunction)`undefined`.

**Returns:** `true` if the comparison passes, or `false` otherwise

- If `attachment` is `undefined` or does not have `aspect`, return `true`.
- If `compareFunc` is `undefined` or ["always"](https://gpuweb.github.io#dom-gpucomparefunction-always)`true`.
- Let `attachmentValue` be the value of `aspect` of `attachment` at `destination`.
- Return `true` if comparing `value` with `attachmentValue` using `compareFunc` succeeds, and `false` otherwise.

Processing of fragments happens in parallel, while any side effects,
such as writes into [GPUBufferBindingType](https://gpuweb.github.io#enumdef-gpubufferbindingtype)["storage"](https://gpuweb.github.io#dom-gpubufferbindingtype-storage)

#### 23.2.7. Output Merging

Output merging is a fixed-function stage of the render [pipeline](https://gpuweb.github.io#pipeline) that
outputs the fragment color, depth and stencil data to be written into the render pass attachments.

**Arguments:**

- `fragment`: The [Fragment](https://gpuweb.github.io#fragment), produced by [§ 23.2.6 Fragment Processing](https://gpuweb.github.io#fragment-processing).
- `pipeline`: The current [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
- `state`: The active [RenderState](https://gpuweb.github.io#renderstate).

1. Let `depthStencilDesc` be `pipeline`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)[depthStencil](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-depthstencil)
2. If `pipeline`.[\[\[writesDepth\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesdepth-slot)`true` and `fragment`.[depthPassed](https://gpuweb.github.io#fragment-depthpassed) is `true`:
  1. Set the value of the depth aspect of `state`.[\[\[depthStencilAttachment\]\]](https://gpuweb.github.io#dom-renderstate-depthstencilattachment-slot)`fragment`.[destination](https://gpuweb.github.io#fragment-destination) to `fragment`.[depth](https://gpuweb.github.io#fragment-depth).
3. If `pipeline`.[\[\[writesStencil\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-writesstencil-slot)
  1. Set `stencilState` to `depthStencilDesc`.[stencilFront](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilfront)`fragment`.[frontFacing](https://gpuweb.github.io#fragment-frontfacing) is `true` and `depthStencilDesc`.[stencilBack](https://gpuweb.github.io#dom-gpudepthstencilstate-stencilback)
  2. If `fragment`.[stencilPassed](https://gpuweb.github.io#fragment-stencilpassed) is `false`:
    - Let `stencilOp` be `stencilState`.[failOp](https://gpuweb.github.io#dom-gpustencilfacestate-failop)
Otherwise, if `fragment`.[depthPassed](https://gpuweb.github.io#fragment-depthpassed) is `false`:
    - Let `stencilOp` be `stencilState`.[depthFailOp](https://gpuweb.github.io#dom-gpustencilfacestate-depthfailop)
Otherwise: 
    - Let `stencilOp` be `stencilState`.[passOp](https://gpuweb.github.io#dom-gpustencilfacestate-passop)
  3. Update the value of the stencil aspect of `state`.[\[\[depthStencilAttachment\]\]](https://gpuweb.github.io#dom-renderstate-depthstencilattachment-slot)`fragment`.[destination](https://gpuweb.github.io#fragment-destination) by performing the operation described by `stencilOp`.

The depth input to this stage, if any, is clamped to the current [\[\[viewport\]\]](https://gpuweb.github.io#dom-renderstate-viewport-slot)`frag_depth` builtin).

**Arguments:**

- `fragment`: The [Fragment](https://gpuweb.github.io#fragment), produced by [§ 23.2.6 Fragment Processing](https://gpuweb.github.io#fragment-processing).
- `pipeline`: The current [GPURenderPipeline](https://gpuweb.github.io#gpurenderpipeline)
- `state`: The active [RenderState](https://gpuweb.github.io#renderstate).

1. If `fragment`.[depthPassed](https://gpuweb.github.io#fragment-depthpassed) is `false` or `fragment`.[stencilPassed](https://gpuweb.github.io#fragment-stencilpassed) is `false`, return.
2. Let `targets` be `pipeline`.[\[\[descriptor\]\]](https://gpuweb.github.io#dom-gpurenderpipeline-descriptor-slot)[fragment](https://gpuweb.github.io#dom-gpurenderpipelinedescriptor-fragment)[targets](https://gpuweb.github.io#dom-gpufragmentstate-targets)
3. For each `attachment` of `state`.[\[\[colorAttachments\]\]](https://gpuweb.github.io#dom-renderstate-colorattachments-slot)
  1. Let `color` be the value from `fragment`.[colors](https://gpuweb.github.io#fragment-colors) that corresponds with `attachment`.
  2. Let `targetDesc` be the `targets` entry that corresponds with `attachment`.
  3. If `targetDesc`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[provided](https://infra.spec.whatwg.org/#map-exists):
    1. Let `colorBlend` be `targetDesc`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[color](https://gpuweb.github.io#dom-gpublendstate-color)
    2. Let `alphaBlend` be `targetDesc`.[blend](https://gpuweb.github.io#dom-gpucolortargetstate-blend)[alpha](https://gpuweb.github.io#dom-gpublendstate-alpha)
    3. Set the RGB components of `color` to the value computed by performing the operation described by `colorBlend`.[operation](https://gpuweb.github.io#dom-gpublendcomponent-operation)`colorBlend`.[srcFactor](https://gpuweb.github.io#dom-gpublendcomponent-srcfactor)`colorBlend`.[dstFactor](https://gpuweb.github.io#dom-gpublendcomponent-dstfactor)
    4. Set the alpha component of `color` to the value computed by performing the operation described by `alphaBlend`.[operation](https://gpuweb.github.io#dom-gpublendcomponent-operation)`alphaBlend`.[srcFactor](https://gpuweb.github.io#dom-gpublendcomponent-srcfactor)`alphaBlend`.[dstFactor](https://gpuweb.github.io#dom-gpublendcomponent-dstfactor)
  4. Set the value of `attachment` at `fragment`.[destination](https://gpuweb.github.io#fragment-destination) to `color`.

#### 23.2.8. No Color Output

In no-color-output mode, [pipeline](https://gpuweb.github.io#pipeline) does not produce any color attachment outputs.

The [pipeline](https://gpuweb.github.io#pipeline) still performs rasterization and produces depth values
based on the vertex position output. The depth testing and stencil operations can still be used.

#### 23.2.9. Alpha to Coverage

In alpha-to-coverage mode, an additional alpha-to-coverage mask
of MSAA samples is generated based on the `alpha` component of the
fragment shader output value at `@location(0)`.

The algorithm of producing the extra mask is platform-dependent and can vary for different pixels. It guarantees that:

- if `alpha` ≤ 0.0, the result is 0x0
- if `alpha` ≥ 1.0, the result is 0xFFFFFFFF
- intermediate `alpha` values should result in a proportionate number of bits set to 1 in the mask.
Not all platforms guarantee that the number of bits set to 1 in the mask monotonically
increases as alpha increases for a given pixel.

#### 23.2.10. Per-Sample Shading

When rendering into multisampled render attachments, fragment shaders can be run once per-pixel or once per-sample.
Fragment shaders **must** run once per-sample if either the `sample_index` [builtin](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values) or `sample` [interpolation sampling](https://gpuweb.github.io/gpuweb/wgsl/#interpolation-sampling)
is used and contributes to the shader output. Otherwise fragment shaders **may** run once per-pixel with the result
broadcast out to each of the samples included in the [final sample mask](https://gpuweb.github.io#final-sample-mask).

When using per-sample shading, the color output for sample `N` is produced by the fragment shader execution
with `sample_index` == `N` for the current pixel.

#### 23.2.11. Sample Masking

The final sample mask for a pixel is computed as:
[rasterization mask](https://gpuweb.github.io#rasterization-mask) & [mask](https://gpuweb.github.io#dom-gpumultisamplestate-mask)[shader-output mask](https://gpuweb.github.io#shader-output-mask).

Only the lower [count](https://gpuweb.github.io#dom-gpumultisamplestate-count)

If the least-significant bit at position `N` of the [final sample mask](https://gpuweb.github.io#final-sample-mask) has value of "0",
the sample color outputs (corresponding to sample `N`) to all attachments of the fragment shader are discarded.
Also, no depth test or stencil operations are executed on the relevant samples of the depth-stencil attachment.

The rasterization mask is produced by the rasterization stage, based on the shape of the rasterized polygon. The samples included in the shape get the relevant bits 1 in the mask.

The shader-output mask takes the output value of "sample\_mask" [builtin](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values) in the fragment shader.
If the builtin is not output from the fragment shader, and [alphaToCoverageEnabled](https://gpuweb.github.io#dom-gpumultisamplestate-alphatocoverageenabled)[shader-output mask](https://gpuweb.github.io#shader-output-mask) becomes the [alpha-to-coverage mask](https://gpuweb.github.io#alpha-to-coverage-mask). Otherwise, it defaults to 0xFFFFFFFF.

## 24. Type Definitions

typedef \[EnforceRange unsigned long GPUBufferDynamicOffset typedef \[EnforceRange unsigned long GPUStencilValue typedef \[EnforceRange unsigned long GPUSampleMask typedef \[EnforceRange long GPUDepthBias typedef \[EnforceRange unsigned long long GPUSize64 typedef \[EnforceRange unsigned long GPUIntegerCoordinate typedef \[EnforceRange unsigned long GPUIndex32 typedef \[EnforceRange unsigned long GPUSize32 typedef \[EnforceRange long GPUSignedOffset32 typedef unsigned long long GPUSize64Out typedef unsigned long GPUIntegerCoordinateOut typedef unsigned long GPUSize32Out typedef unsigned long GPUFlagsConstant 

### 24.1. Colors & Vectors

dictionary GPUColorDict required double r required double g required double b required double a typedef (sequence double or GPUColorDict GPUColor 

Note: `double` is large enough to precisely hold 32-bit signed/unsigned
integers and single-precision floats.

- `r`,  of type [double](https://webidl.spec.whatwg.org/#idl-double)
- The red channel value.
- `g`,  of type [double](https://webidl.spec.whatwg.org/#idl-double)
- The green channel value.
- `b`,  of type [double](https://webidl.spec.whatwg.org/#idl-double)
- The blue channel value.
- `a`,  of type [double](https://webidl.spec.whatwg.org/#idl-double)
- The alpha channel value.

[GPUColor](https://gpuweb.github.io#typedefdef-gpucolor)
`color`, depending on its type, the syntax:

- `color`.r refers to
either [GPUColorDict](https://gpuweb.github.io#dictdef-gpucolordict)[r](https://gpuweb.github.io#dom-gpucolordict-r)[asserting](https://infra.spec.whatwg.org/#assert) there is such an item).
- `color`.g refers to
either [GPUColorDict](https://gpuweb.github.io#dictdef-gpucolordict)[g](https://gpuweb.github.io#dom-gpucolordict-g)[asserting](https://infra.spec.whatwg.org/#assert) there is such an item).
- `color`.b refers to
either [GPUColorDict](https://gpuweb.github.io#dictdef-gpucolordict)[b](https://gpuweb.github.io#dom-gpucolordict-b)[asserting](https://infra.spec.whatwg.org/#assert) there is such an item).
- `color`.a refers to
either [GPUColorDict](https://gpuweb.github.io#dictdef-gpucolordict)[a](https://gpuweb.github.io#dom-gpucolordict-a)[asserting](https://infra.spec.whatwg.org/#assert) there is such an item).

**Arguments:**

- `color`: The [GPUColor](https://gpuweb.github.io#typedefdef-gpucolor)

**Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)

[Content timeline](https://gpuweb.github.io#content-timeline) steps:

dictionary GPUOrigin2DDict GPUIntegerCoordinate x GPUIntegerCoordinate y typedef (sequence GPUIntegerCoordinate or GPUOrigin2DDict GPUOrigin2D 

[GPUOrigin2D](https://gpuweb.github.io#typedefdef-gpuorigin2d)
`origin`, depending on its type, the syntax:

- `origin`.x refers to
either [GPUOrigin2DDict](https://gpuweb.github.io#dictdef-gpuorigin2ddict)[x](https://gpuweb.github.io#dom-gpuorigin2ddict-x)
- `origin`.y refers to
either [GPUOrigin2DDict](https://gpuweb.github.io#dictdef-gpuorigin2ddict)[y](https://gpuweb.github.io#dom-gpuorigin2ddict-y)

**Arguments:**

- `origin`: The [GPUOrigin2D](https://gpuweb.github.io#typedefdef-gpuorigin2d)

**Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)

[Content timeline](https://gpuweb.github.io#content-timeline) steps:

dictionary GPUOrigin3DDict GPUIntegerCoordinate x GPUIntegerCoordinate y GPUIntegerCoordinate z typedef (sequence GPUIntegerCoordinate or GPUOrigin3DDict GPUOrigin3D 

[GPUOrigin3D](https://gpuweb.github.io#typedefdef-gpuorigin3d)
`origin`, depending on its type, the syntax:

- `origin`.x refers to
either [GPUOrigin3DDict](https://gpuweb.github.io#dictdef-gpuorigin3ddict)[x](https://gpuweb.github.io#dom-gpuorigin3ddict-x)
- `origin`.y refers to
either [GPUOrigin3DDict](https://gpuweb.github.io#dictdef-gpuorigin3ddict)[y](https://gpuweb.github.io#dom-gpuorigin3ddict-y)
- `origin`.z refers to
either [GPUOrigin3DDict](https://gpuweb.github.io#dictdef-gpuorigin3ddict)[z](https://gpuweb.github.io#dom-gpuorigin3ddict-z)

**Arguments:**

- `origin`: The [GPUOrigin3D](https://gpuweb.github.io#typedefdef-gpuorigin3d)

**Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)

[Content timeline](https://gpuweb.github.io#content-timeline) steps:

dictionary GPUExtent3DDict required GPUIntegerCoordinate width GPUIntegerCoordinate height GPUIntegerCoordinate depthOrArrayLayers typedef (sequence GPUIntegerCoordinate or GPUExtent3DDict GPUExtent3D 

- `width`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate)
- The width of the extent.
- `height`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate), defaulting to `1`
- The height of the extent.
- `depthOrArrayLayers`,  of type [GPUIntegerCoordinate](https://gpuweb.github.io#typedefdef-gpuintegercoordinate), defaulting to `1`
- The depth of the extent or the number of array layers it contains.
If used with a [GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureDimension](https://gpuweb.github.io#enumdef-gputexturedimension)["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)[GPUTexture](https://gpuweb.github.io#gputexture)[GPUTextureDimension](https://gpuweb.github.io#enumdef-gputexturedimension)["2d"](https://gpuweb.github.io#dom-gputexturedimension-2d)

[GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)
`extent`, depending on its type, the syntax:

- `extent`.width refers to
either [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)[width](https://gpuweb.github.io#dom-gpuextent3ddict-width)[asserting](https://infra.spec.whatwg.org/#assert) there is such an item).
- `extent`.height refers to
either [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)[height](https://gpuweb.github.io#dom-gpuextent3ddict-height)
- `extent`.depthOrArrayLayers refers to
either [GPUExtent3DDict](https://gpuweb.github.io#dictdef-gpuextent3ddict)[depthOrArrayLayers](https://gpuweb.github.io#dom-gpuextent3ddict-depthorarraylayers)

**Arguments:**

- `extent`: The [GPUExtent3D](https://gpuweb.github.io#typedefdef-gpuextent3d)

**Returns:** [undefined](https://webidl.spec.whatwg.org/#idl-undefined)

[Content timeline](https://gpuweb.github.io#content-timeline) steps:

1. Throw a [TypeError](https://webidl.spec.whatwg.org/#exceptiondef-typeerror)

## 25. Feature Index

*This section is non-normative.*

This section describes what is enabled by each feature in [GPUFeatureName](https://gpuweb.github.io#gpufeaturename)

Dependencies between features are described in [§ 3.6.1.2 Feature Dependencies](https://gpuweb.github.io#feature-dependencies) and enforced in
[§ 4.2.1 Adapter Capability Guarantees](https://gpuweb.github.io#adapter-capability-guarantees) and [a new device](https://gpuweb.github.io#a-new-device).

### 25.1. `"core-features-and-limits"`

When enabled:

- Lifts extra validation rules that are specific to WebGPU ["compatibility"](https://gpuweb.github.io#feature-level-string-compatibility) mode.
These are not enumerated here; see references to ["core-features-and-limits"](https://gpuweb.github.io#core-features-and-limits)
- Ensures all limits are set to their [default](https://gpuweb.github.io#limit-default) value
(rather than their [compatibility mode default](https://gpuweb.github.io#limit-compatibility-mode-default) value), or [better](https://gpuweb.github.io#limit-better) if requested.

Note:
This feature is always available when calling [requestAdapter()](https://gpuweb.github.io#dom-gpu-requestadapter)[featureLevel](https://gpuweb.github.io#dom-gpurequestadapteroptions-featurelevel)["compatibility"](https://gpuweb.github.io#feature-level-string-compatibility),
in which case its availability depends on the system. See those definitions for details.

### 25.2. `"depth-clip-control"`

When enabled:

- Allows [depth clipping](https://gpuweb.github.io#depth-clipping) to be disabled.

New [GPUPrimitiveState](https://gpuweb.github.io#dictdef-gpuprimitivestate)

### 25.3. `"depth32float-stencil8"`

When enabled:

- Allows for explicit creation of textures of format ["depth32float-stencil8"](https://gpuweb.github.io#dom-gputextureformat-depth32float-stencil8)

New [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

### 25.4. `"texture-compression-bc"`

When enabled:

- Allows for explicit creation of textures of [BC compressed formats](https://registry.khronos.org/DataFormat/specs/1.3/dataformat.1.3.html#_compressed_texture_image_formats) which include the "S3TC",
"RGTC", and "BPTC" formats.
Note:
This feature only supports 2D textures. If 3D textures are needed, use
["texture-compression-bc-sliced-3d"](https://gpuweb.github.io#texture-compression-bc-sliced-3d)

New [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

### 25.5. `"texture-compression-bc-sliced-3d"`

When enabled:

- Allows the [3d](https://gpuweb.github.io#dom-gputexturedimension-3d)[BC compressed formats](https://registry.khronos.org/DataFormat/specs/1.3/dataformat.1.3.html#_compressed_texture_image_formats).

### 25.6. `"texture-compression-etc2"`

When enabled:

- Allows for explicit creation of textures of [ETC2 compressed formats](https://registry.khronos.org/DataFormat/specs/1.3/dataformat.1.3.html#ETC2).
Note: This feature only supports 2D textures.

New [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

### 25.7. `"texture-compression-astc"`

When enabled:

- Allows for explicit creation of textures of [ASTC compressed formats](https://registry.khronos.org/DataFormat/specs/1.3/dataformat.1.3.html#ASTC).
Note:
This feature only supports 2D textures. If 3D textures are needed, use
["texture-compression-astc-sliced-3d"](https://gpuweb.github.io#texture-compression-astc-sliced-3d)

New [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

### 25.8. `"texture-compression-astc-sliced-3d"`

When enabled:

- Allows the [3d](https://gpuweb.github.io#dom-gputexturedimension-3d)[ASTC compressed formats](https://registry.khronos.org/DataFormat/specs/1.3/dataformat.1.3.html#ASTC).

### 25.9. `"timestamp-query"`

When enabled:

- Adds the ability to query timestamps from GPU command buffers. See [§ 20.4 Timestamp Query](https://gpuweb.github.io#timestamp).

New [GPUQueryType](https://gpuweb.github.io#enumdef-gpuquerytype)

New [GPUComputePassDescriptor](https://gpuweb.github.io#dictdef-gpucomputepassdescriptor)

New [GPURenderPassDescriptor](https://gpuweb.github.io#dictdef-gpurenderpassdescriptor)

### 25.10. `"indirect-first-instance"`

When enabled:

- Allows the use of non-zero `firstInstance` values in [indirect draw parameters](https://gpuweb.github.io#indirect-draw-parameters) and [indirect drawIndexed parameters](https://gpuweb.github.io#indirect-drawindexed-parameters).

### 25.11. `"shader-f16"`

When enabled:

- Allows the use of the half-precision floating-point type [f16](https://gpuweb.github.io/gpuweb/wgsl/#f16) in WGSL.

New WGSL extensions:

### 25.12. `"rg11b10ufloat-renderable"`

When enabled:

- Allows the [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)["rg11b10ufloat"](https://gpuweb.github.io#dom-gputextureformat-rg11b10ufloat)
Implicitly, this allows ["rg11b10ufloat"](https://gpuweb.github.io#dom-gputextureformat-rg11b10ufloat)[copyExternalImageToTexture()](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture)

### 25.13. `"bgra8unorm-storage"`

When enabled:

- Allows the [STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)["bgra8unorm"](https://gpuweb.github.io#dom-gputextureformat-bgra8unorm)

### 25.14. `"float32-filterable"`

When enabled:

- Makes textures with the following formats [filterable](https://gpuweb.github.io#filterable):

### 25.15. `"float32-blendable"`

When enabled:

- Makes textures with the following formats [blendable](https://gpuweb.github.io#blendable):

### 25.16. `"clip-distances"`

When enabled:

- Allows the use of [clip\_distances](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-clip_distances) in WGSL.

New WGSL extensions:

### 25.17. `"dual-source-blending"`

When enabled:

- Allows the use of [blend\_src](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) in WGSL and simultaneously using both pixel shader outputs
(`@blend_src(0)` and `@blend_src(1)`) as inputs to a blending operation with the single color
attachment at [location](https://gpuweb.github.io/gpuweb/wgsl/#input-output-locations) `0`.

New [GPUBlendFactor](https://gpuweb.github.io#enumdef-gpublendfactor)

New WGSL extensions:

### 25.18. `"subgroups"`

When enabled:

- Allows the use of the subgroup and quad operations in WGSL.
- The following entries of [GPUAdapterInfo](https://gpuweb.github.io#gpuadapterinfo)

New WGSL extensions:

### 25.19. `"texture-formats-tier1"`

When enabled:

- Enables ["rg11b10ufloat-renderable"](https://gpuweb.github.io#rg11b10ufloat-renderable)
- Supports the below new [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[blendable](https://gpuweb.github.io#blendable), `multisampling` capabilities and the [STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)["read-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-only)["write-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-write-only)[GPUStorageTextureAccess](https://gpuweb.github.io#enumdef-gpustoragetextureaccess)
Implicitly, this allows the following new destination formats in [copyExternalImageToTexture()](https://gpuweb.github.io#dom-gpuqueue-copyexternalimagetotexture)
- Supports the [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[blendable](https://gpuweb.github.io#blendable), `multisampling` and `resolve` capabilities on below [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)
- Supports the [STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)["read-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-only)["write-only"](https://gpuweb.github.io#dom-gpustoragetextureaccess-write-only)[GPUStorageTextureAccess](https://gpuweb.github.io#enumdef-gpustoragetextureaccess)[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

### 25.20. `"texture-formats-tier2"`

When enabled:

- Enables ["texture-formats-tier1"](https://gpuweb.github.io#texture-formats-tier1)
- Allows the ["read-write"](https://gpuweb.github.io#dom-gpustoragetextureaccess-read-write)[GPUStorageTextureAccess](https://gpuweb.github.io#enumdef-gpustoragetextureaccess)[GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)

### 25.21. `"primitive-index"`

When enabled:

- Allows the use of [primitive\_index](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-primitive_index) in WGSL.

New WGSL extensions:

### 25.22. `"texture-component-swizzle"`

When enabled:

- Allows [GPUTextureView](https://gpuweb.github.io#gputextureview)[TEXTURE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-texture_binding)
- Defines previously-implementation-defined behavior when [§ 26.1.2.1 Reading and Sampling Depth/Stencil Textures](https://gpuweb.github.io#reading-depth-stencil).

New [GPUTextureViewDescriptor](https://gpuweb.github.io#dictdef-gputextureviewdescriptor)

### 25.23. `"subgroup-size-control"`

When enabled:

- Enables ["subgroups"](https://gpuweb.github.io#subgroups)
- Allows the use of the [subgroup\_size](https://gpuweb.github.io/gpuweb/wgsl/#subgroup-size-attr) attribute in WGSL to control compute pipeline subgroup size.

New WGSL extensions:

### 25.24. `"texture-compression-unaligned"`

When enabled:

- Allows the creation of textures with block-based compressed [GPUTextureFormat](https://gpuweb.github.io#enumdef-gputextureformat)[size](https://gpuweb.github.io#dom-gputexturedescriptor-size)[texel block width](https://gpuweb.github.io#texel-block-width) and [texel block height](https://gpuweb.github.io#texel-block-height).

### 25.25. `"atomic-vec2u-min-max"`

When enabled:

- Allows the use of 64-bit atomic minimum and maximum operations on `atomic<vec2u>` in WGSL.

New WGSL extensions:

## 26. Appendices

### 26.1. Texture Format Capabilities

#### 26.1.1. Plain color formats

All [supported](https://gpuweb.github.io#abstract-opdef-validate-texture-format-required-features) plain color formats support usages
[COPY\_SRC](https://gpuweb.github.io#dom-gputextureusage-copy_src)[COPY\_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)[TEXTURE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-texture_binding)["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)

The [RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)[GPUTextureUsage.RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)[GPUTextureUsage.STORAGE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-storage_binding)

The render target pixel byte cost
and render target component alignment
are used to validate the [maxColorAttachmentBytesPerSample](https://gpuweb.github.io#dom-supported-limits-maxcolorattachmentbytespersample)

Note:
The [texel block memory cost](https://gpuweb.github.io#texel-block-memory-cost) of each of these formats is the same as its
[texel block copy footprint](https://gpuweb.github.io#texel-block-copy-footprint).

#### 26.1.2. Depth-stencil formats

A depth-or-stencil format is any format with depth and/or stencil aspects.
A combined depth-stencil format is a [depth-or-stencil format](https://gpuweb.github.io#depth-or-stencil-format) that has both
depth and stencil aspects.

All [depth-or-stencil formats](https://gpuweb.github.io#depth-or-stencil-format) support the [COPY\_SRC](https://gpuweb.github.io#dom-gputextureusage-copy_src)[COPY\_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)[TEXTURE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-texture_binding)[RENDER\_ATTACHMENT](https://gpuweb.github.io#dom-gputextureusage-render_attachment)["3d"](https://gpuweb.github.io#dom-gputexturedimension-3d)

Depth textures cannot be used with ["filtering"](https://gpuweb.github.io#dom-gpusamplerbindingtype-filtering)["comparison"](https://gpuweb.github.io#dom-gpusamplerbindingtype-comparison)

| Format | NOTE: [Texel block memory cost](https://gpuweb.github.io#texel-block-memory-cost) (Bytes) | Aspect | [GPUTextureSampleType](https://gpuweb.github.io#enumdef-gputexturesampletype) | Valid [texel copy](https://gpuweb.github.io#texel-copy) source | Valid [texel copy](https://gpuweb.github.io#texel-copy) destination | [Texel block copy footprint](https://gpuweb.github.io#texel-block-copy-footprint) (Bytes) | Aspect-specific format | 
|---|---|---|---|---|---|---|---|
| [stencil8](https://gpuweb.github.io#dom-gputextureformat-stencil8) | 1 − 4 | stencil | ["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint) | ✓ |  | 1 | [stencil8](https://gpuweb.github.io#dom-gputextureformat-stencil8) | 
| [depth16unorm](https://gpuweb.github.io#dom-gputextureformat-depth16unorm) | 2 | depth | ["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth), ["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float) | ✓ |  | 2 | [depth16unorm](https://gpuweb.github.io#dom-gputextureformat-depth16unorm) | 
| [depth24plus](https://gpuweb.github.io#dom-gputextureformat-depth24plus) | 4 | depth | ["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth), ["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float) | ✗ |  | – | [depth24plus](https://gpuweb.github.io#dom-gputextureformat-depth24plus) | 
| [depth24plus-stencil8](https://gpuweb.github.io#dom-gputextureformat-depth24plus-stencil8) | 4 − 8 | depth | ["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth), ["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float) | ✗ |  | – | [depth24plus](https://gpuweb.github.io#dom-gputextureformat-depth24plus) | 
|  |  | stencil | ["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint) | ✓ |  | 1 | [stencil8](https://gpuweb.github.io#dom-gputextureformat-stencil8) | 
| [depth32float](https://gpuweb.github.io#dom-gputextureformat-depth32float) | 4 | depth | ["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth), ["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float) | ✓ | ✗ | 4 | [depth32float](https://gpuweb.github.io#dom-gputextureformat-depth32float) | 
| [depth32float-stencil8](https://gpuweb.github.io#dom-gputextureformat-depth32float-stencil8) | 5 − 8 | depth | ["depth"](https://gpuweb.github.io#dom-gputexturesampletype-depth), ["unfilterable-float"](https://gpuweb.github.io#dom-gputexturesampletype-unfilterable-float) | ✓ | ✗ | 4 | [depth32float](https://gpuweb.github.io#dom-gputextureformat-depth32float) | 
|  |  | stencil | ["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint) | ✓ |  | 1 | [stencil8](https://gpuweb.github.io#dom-gputextureformat-stencil8) | 

24-bit depth refers to a 24-bit unsigned normalized depth format with a range from 0.0 to 1.0, which would be spelled "depth24unorm" if exposed.

##### 26.1.2.1. Reading and Sampling Depth/Stencil Textures

It is [possible](https://gpuweb.github.io#abstract-opdef-validating-shader-binding) to bind a depth-aspect [GPUTextureView](https://gpuweb.github.io#gputextureview)`texture_depth_*` binding or a binding with other non-depth 2d/cube texture types.

A stencil-aspect [GPUTextureView](https://gpuweb.github.io#gputextureview)[sampleType](https://gpuweb.github.io#dom-gputexturebindinglayout-sampletype)[GPUBindGroupLayout](https://gpuweb.github.io#gpubindgrouplayout)["uint"](https://gpuweb.github.io#dom-gputexturesampletype-uint)

If the ["texture-component-swizzle"](https://gpuweb.github.io#dom-gpufeaturename-texture-component-swizzle)`(V, 0, 0, 1)`
where V is the actual depth or stencil value. Otherwise, the values are `(V, X, X, X)` where each X
is an [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined) unspecified value.

To reduce compatibility issues in practice, implementations **should** provide `(V, 0, 0, 1)`
wherever possible, even if the ["texture-component-swizzle"](https://gpuweb.github.io#dom-gpufeaturename-texture-component-swizzle)

For depth-aspect bindings, the unspecified values are not visible through bindings with
`texture_depth_*` types.

[If a depth texture is bound to](https://gpuweb.github.io#example-cb229a6b)

`tex` with type `texture_2d<f32>`:
    - `textureSample(tex, ...)` will return `vec4<f32>(D, X, X, X)`.
- `textureGather(0, tex, ...)` will return `vec4<f32>(D1, D2, D3, D4)`.
- `textureGather(2, tex, ...)` will return `vec4<f32>(X1, X2, X3, X4)` (a completely unspecified value).

Note:
Short of adding a new more constrained stencil sampler type (like depth), it’s infeasible for
implementations to efficiently paper over the driver differences for depth/stencil reads.
As this was not a portability pain point for WebGL, it’s not expected to be problematic in WebGPU.
In practice, expect either `(V, V, V, V)` or `(V, 0, 0, 1)` (where `V` is the depth or stencil
value), depending on hardware.

##### 26.1.2.2. Copying Depth/Stencil Textures

The depth aspects of depth32float formats
(["depth32float"](https://gpuweb.github.io#dom-gputextureformat-depth32float)["depth32float-stencil8"](https://gpuweb.github.io#dom-gputextureformat-depth32float-stencil8)

The depth aspects of depth24plus formats
(["depth24plus"](https://gpuweb.github.io#dom-gputextureformat-depth24plus)["depth24plus-stencil8"](https://gpuweb.github.io#dom-gputextureformat-depth24plus-stencil8)[24-bit depth](https://gpuweb.github.io#24-bit-depth) or ["depth32float"](https://gpuweb.github.io#dom-gputextureformat-depth32float)[texel copies](https://gpuweb.github.io#texel-copy) are not allowed with these formats.

- All of these formats can be written in a render pass using a fragment shader that outputs
depth values via the `frag_depth` output.
- Textures with "depth24plus" formats can be read as shader textures, and written to a texture (as a render pass attachment) or buffer (via a storage buffer binding in a compute shader).

#### 26.1.3. Packed formats

All packed texture formats support [COPY\_SRC](https://gpuweb.github.io#dom-gputextureusage-copy_src)[COPY\_DST](https://gpuweb.github.io#dom-gputextureusage-copy_dst)[TEXTURE\_BINDING](https://gpuweb.github.io#dom-gputextureusage-texture_binding)[filterable](https://gpuweb.github.io#filterable).
None of these formats are [renderable](https://gpuweb.github.io#renderable) or support multisampling.

A compressed format is any format with a block size greater than 1×1.

Note:
The [texel block memory cost](https://gpuweb.github.io#texel-block-memory-cost) of each of these formats is the same as its
[texel block copy footprint](https://gpuweb.github.io#texel-block-copy-footprint).

Source: [https://gpuweb.github.io/gpuweb/](<https://gpuweb.github.io/gpuweb/>)
