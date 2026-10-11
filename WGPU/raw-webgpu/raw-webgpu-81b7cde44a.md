&lt;p&gt;See the Pen &amp;lt;a href=&amp;#x27;&lt;a href="https://codepen.io/alaingalvan/pen/GRgvLGw&amp;#x27;&amp;amp;gt;Raw"&gt;https://codepen.io/alaingalvan/pen/GRgvLGw&amp;#x27;&amp;amp;gt;Raw&lt;/a&gt;
WebGPU&amp;lt;/a&amp;gt; by Alain Galvan (&amp;lt;a
href=&amp;#x27;&lt;a href="https://codepen.io/alaingalvan&amp;#x27;&amp;amp;gt;@alaingalvan&amp;amp;lt;/a"&gt;https://codepen.io/alaingalvan&amp;#x27;&amp;amp;gt;@alaingalvan&amp;amp;lt;/a&lt;/a&gt;&amp;gt;) on &amp;lt;a
href=&amp;#x27;&lt;a href="https://codepen.io&amp;#x27;&amp;amp;gt;CodePen&amp;amp;lt;/a"&gt;https://codepen.io&amp;#x27;&amp;amp;gt;CodePen&amp;amp;lt;/a&lt;/a&gt;&amp;gt;.&lt;/p&gt;

![WebGPU Logo by WebGPU Team](<raw-webgpu/63648a1f63748538c084d40a.svg>)

WebGPU is a new graphics API for the web that follows the architecture of [modern computer graphics APIs](https://alain.xyz/blog/comparison-of-modern-graphics-apis) such as Microsoft [DirectX 12](https://alain.xyz/blog/raw-directx12), [Vulkan](https://alain.xyz/blog/raw-vulkan), and Apple [Metal](https://alain.xyz/blog/raw-metal). This shift in paradigm for web graphics APIs allows users to take advantage of the same benefits native graphics APIs bring: faster applications thanks to the ability to keep the GPU busy with work, less graphics driver specific bugs, and the potential for new features should they be implemented in the future.

WebGPU is arguably the most complex out of all rendering APIs on the web, though this cost is offset by both the increase in performance and guarantee of future support that the API provides. This post aims to demystify the API, making it easier to piece together how to write web apps that use it.

---

I've prepared a [Github repo](https://github.com/alaingalvan/webgpu-seed) with everything you need to get started. We'll walk through writing a WebGPU *Hello Triangle* application in [TypeScript](https://www.typescriptlang.org/).

> Check out my other post on [WebGL](https://alain.xyz/blog/raw-webgl) for writing graphics applications with an older but widely supported web graphics API.

<a id="setup"></a>

## Setup

WebGPU is supported on a [wide range of platforms](https://caniuse.com/?search=webgpu):

- [Google Chrome](https://www.google.com/chrome/): WebGPU is available on the latest release of Chrome.
- [Mozilla Firefox](https://www.mozilla.org/en-US/firefox/channel/desktop/): WebGPU is supported but you may need to visit `about:config` and enable `dom.webgpu.enabled`.
- [Microsoft Edge](https://www.microsoftedgeinsider.com/en-us/download): WebGPU is functionally the same as Chrome.
- Apple Safari: Can be enabled with the `WebGPU` feature flag.

Once you have a browser that features WebGPU, install the following:

- [Git](https://git-scm.com/)
- [Node.js](https://nodejs.org/en/)
- A Text Editor such as [Visual Studio Code](https://code.visualstudio.com/).

Then type the following in any terminal such as [VS Code's Integrated Terminal](https://code.visualstudio.com/docs/editor/integrated-terminal).

```
# 🐑 Clone the repo
git clone https://github.com/alaingalvan/webgpu-seed

# 💿 go inside the folder
cd webgpu-seed

# 🔨 Start building the project
npm start
```

> Refer to [this blog post on designing web libraries and apps](https://alain.xyz/blog/designing-a-web-app) for more details on Node.js, packages, etc.

<a id="project-layout"></a>

## Project Layout

As your project becomes more complex, you'll want to separate files and organize your application to something more akin to a game or renderer, check out this post on [game engine architecture](https://alain.xyz/blog/game-engine-architecture) and this one on [real time renderer architecture](https://alain.xyz/blog/realtime-renderer-architectures) for more details.

```
├─ 📂 node_modules/   # 👶 Dependencies
│  ├─ 📁 gl-matrix      # ➕ Linear Algebra
│  └─ 📁 ...            # 🕚 Other Dependencies (TypeScript, Webpack, etc.)
├─ 📂 src/            # 🌟 Source Files
│  ├─ 📄 renderer.ts    # 🔺 Triangle Renderer
│  └─ 📄 main.ts        # 🏁 Application Main
├─ 📄 .gitignore      # 👁️ Ignore Certain Files in Git Repo
├─ 📄 package.json    # 📦 Node Package File
├─ 📄 license.md      # ⚖️ Your License (Unlicense)
└─ 📃readme.md        # 📖 Read Me!
```

<a id="dependencies"></a>

### Dependencies

- [gl-matrix](https://github.com/toji/gl-matrix) - A JavaScript library that allows users to write `glsl` like JavaScript code, with types for vectors, matrices, etc. While not in use in this sample, it's incredibly useful for programming more advanced topics such as camera matrices.
- [TypeScript](https://github.com/microsoft/typescript) - JavaScript with types, makes it significantly easier to program web apps with instant autocomplete and type checking.
- [Webpack](https://github.com/webpack/webpack) - A JavaScript compilation tool to build minified outputs and test our apps faster.

<a id="overview"></a>

## Overview

In this application we will need to do the following:

1. **Initialize the API** - Check if `navigator.gpu` exists, and if it does, request a `GPUAdapter`, then request a `GPUDevice`, and get that device's default `GPUQueue`.
2. **Setup Frame Backings** - create a `GPUCanvasContext` and configure it to receive a `GPUTexture` for the current frame, as well as any other attachments you might need (such as a depth-stencil texture, etc.). Create `GPUTextureView`s for those textures.
3. **Initialize Resources** - Create your Vertex and Index `GPUBuffer`s, load your WebGPU Shading Language (WGSL) shaders as `GPUShaderModule`s, create your `GPURenderPipeline` by describing every stage of the graphics pipeline. Finally, build your `GPUCommandEncoder` with what what render passes you intend to run, then a `GPURenderPassEncoder` with all the draw calls you intend to execute for that render pass.
4. **Render** - Submit your `GPUCommandEncoder` by calling `.finish()`, and submitting that to your `GPUQueue`. Refresh the canvas context by calling `requestAnimationFrame`.
5. **Destroy** - Destroy any data structures after you're done using the API.

The following will explain snippets from that can be found in the [Github repo](https://github.com/alaingalvan/webgpu-seed), with certain parts omitted, and *member variables (`this.memberVariable`) declared inline without the `this.` prefix so their type is easier to see and the examples here can work on their own*.

<a id="initialize-api"></a>

## Initialize API

<a id="entry-point"></a>

### Entry Point

![Entry Point Diagram](<raw-webgpu/87b18594c05d2c7d46100ed7.svg>)

To access the WebGPU API, you need to see if there exists a `gpu` object in the global `navigator`.

```
Checking if WebGPU is Supported// 🏭 Entry to WebGPUconst entry: GPU = navigator.gpu;if (!entry) {  throw new Error("WebGPU is not supported on this browser.");}
```

<a id="adapter"></a>

### Adapter

![Adapter Diagram](<raw-webgpu/d02571765fc377dd6326784f.svg>)

An **Adapter** describes the physical properties of a given GPU, such as its name, extensions, and device limits.

```
Finding a GPUAdapter// ✋ Declare adapter handlelet adapter: GPUAdapter = null; // 🙏 Inside an async function... // 🔌 Physical Device Adapteradapter = await entry.requestAdapter();
```

<a id="device"></a>

### Device

![Device Diagram](<raw-webgpu/af433ca8c87e2caeb9144add.svg>)

A **Device** is how you access the *core* of the WebGPU API, and will allow you to create the data structures you'll need.

```
Finding a GPUDevice// ✋ Declare device handlelet device: GPUDevice = null; // 🙏 Inside an async function... // 💻 Logical Devicedevice = await adapter.requestDevice();
```

<a id="queue"></a>

### Queue

![Queue Diagram](<raw-webgpu/2f61d29e8f0f1de0a41038eb.svg>)

A **Queue** allows you to send work asynchronously to the GPU. As of the writing of this post, you can only access a single default queue from a given `GPUDevice`.

```
Finding a GPUQueue// ✋ Declare queue handlelet queue: GPUQueue = null; // 📦 Queuequeue = device.queue;
```

<a id="frame-backings"></a>

## Frame Backings

<a id="canvas-context"></a>

### Canvas Context

![Canvas Element Diagram](<raw-webgpu/bdba9eb9859a21d570a69476.svg>)

In order to see what you're drawing, you'll need an `HTMLCanvasElement` and to setup a **Canvas Context** from that canvas. A Canvas Context manages a series of textures you'll use to present your final render output to your `<canvas>` element.

```
Creating a WebGPU Context// ✋ Declare context handleconst context: GPUCanvasContext = null; // ⚪ Create Contextcontext = canvas.getContext("webgpu"); // ⛓️ Configure Contextconst canvasConfig: GPUCanvasConfiguration = {  device: this.device,  format: "bgra8unorm",  usage: GPUTextureUsage.RENDER_ATTACHMENT | GPUTextureUsage.COPY_SRC,  alphaMode: "opaque",}; context.configure(canvasConfig);
```

<a id="frame-buffer-attachments"></a>

### Frame Buffer Attachments

![Texture Attachments Diagram](<raw-webgpu/c531de19d5fcc62de112715d.svg>)

When executing different passes of your rendering system, you'll need output textures to write to, be it depth textures for depth testing or shadows, or attachments for various aspects of a deferred renderer such as view space normals, PBR reflectivity/roughness, etc.

Frame buffers attachments are references to texture views, which you'll see later when we write our rendering logic.

```
Creating Frame Buffer Attachments// ✋ Declare attachment handleslet depthTexture: GPUTexture = null;let depthTextureView: GPUTextureView = null; // 🤔 Create Depth Backingconst depthTextureDesc: GPUTextureDescriptor = {  size: [canvas.width, canvas.height, 1],  dimension: "2d",  format: "depth24plus-stencil8",  usage: GPUTextureUsage.RENDER_ATTACHMENT | GPUTextureUsage.COPY_SRC,}; depthTexture = device.createTexture(depthTextureDesc);depthTextureView = depthTexture.createView(); // ✋ Declare canvas context image handleslet colorTexture: GPUTexture = null;let colorTextureView: GPUTextureView = null; colorTexture = context.getCurrentTexture();colorTextureView = colorTexture.createView();
```

<a id="initialize-resources"></a>

## Initialize Resources

<a id="vertex--index-buffers"></a>

### Vertex & Index Buffers

![Buffers Diagram](<raw-webgpu/4decf2517f1e92cc51a1e77b.svg>)

A **Buffer** is an array of data, such as a mesh's positional data, color data, index data, etc. When rendering triangles with a raster based *graphics pipeline*, you'll need 1 or more buffers of vertex data (commonly referred to as *Vertex Buffer Objects* or *VBO*s), and 1 buffer of the indices that correspond with each triangle vertex that you intend to draw (otherwise known as an *Index Buffer Object* or *IBO*).

```
Creating GPUBuffers// 📈 Position Vertex Buffer Dataconst positions = new Float32Array([  1.0, -1.0, 0.0, -1.0, -1.0, 0.0, 0.0, 1.0, 0.0,]); // 🎨 Color Vertex Buffer Dataconst colors = new Float32Array([  1.0, 0.0, 0.0, // 🔴  0.0, 1.0, 0.0, // 🟢  0.0, 0.0, 1.0, // 🔵]); // 📇 Index Buffer Dataconst indices = new Uint16Array([0, 1, 2]); // ✋ Declare buffer handleslet positionBuffer: GPUBuffer = null;let colorBuffer: GPUBuffer = null;let indexBuffer: GPUBuffer = null; // 👋 Helper function for creating GPUBuffer(s) out of Typed Arraysconst createBuffer = (arr: Float32Array | Uint16Array, usage: number) => {  // 📏 Align to 4 bytes (thanks @chrimsonite)  let desc = {    size: (arr.byteLength + 3) & ~3,    usage,    mappedAtCreation: true,  };  let buffer = device.createBuffer(desc);   const writeArray =    arr instanceof Uint16Array      ? new Uint16Array(buffer.getMappedRange())      : new Float32Array(buffer.getMappedRange());  writeArray.set(arr);  buffer.unmap();  return buffer;}; positionBuffer = createBuffer(positions, GPUBufferUsage.VERTEX);colorBuffer = createBuffer(colors, GPUBufferUsage.VERTEX);indexBuffer = createBuffer(indices, GPUBufferUsage.INDEX);
```

<a id="shaders"></a>

### Shaders

![WGSL Shaders](<raw-webgpu/6f1c511979c0368951af6754.svg>)

With WebGPU comes a new shader language: **WebGPU Shading Language (WGSL)**:

WebGPU Shading Language is similar to other languages like Rust, Metal Shading Language (MSL), and DirectX High Level Shading Language, with JavaScript style decorators such as `@location(0)`, Rust style code formatting with `snake_case` functions/members, `CamelCase` structs, and functions following `fn my_func() -> i32`:

Here's the vertex shader source:

```
Basic Vertex Shaderstruct VSOut {    @builtin(position) nds_position: vec4<f32>,    @location(0) color: vec3<f32>,}; @vertexfn main(@location(0) in_pos: vec3<f32>,        @location(1) in_color: vec3<f32>) -> VSOut {    var vs_out: VSOut;    vs_out.nds_position = vec4<f32>(in_pos, 1.0);    vs_out.color = inColor;    return vsOut;}
```

Here's the fragment shader source:

```
Basic Fragment Shader@fragmentfn main(@location(0) in_color: vec3<f32>) -> @location(0) vec4<f32> {    return vec4<f32>(in_color, 1.0);}
```

> It is possible to transpile other shading languages like HLSL or GLSL to WGSL using a transpiler like [Mozilla Naga](https://github.com/gfx-rs/naga) or [Google Tint](https://dawn.googlesource.com/tint/), but the process is a bit involved.

<a id="shader-modules"></a>

### Shader Modules

![Shader Modules Diagram](<raw-webgpu/ee1dc5c8808ab8d4fe18774b.svg>)

**Shader Module**s are *plain text WGSL files* that execute on the GPU when executing a given pipeline.

```
Creating Shader Modules// 📄 dDeclare in line your WGSL code or import if using Webpack asset/source:import vertShaderCode from "./shaders/triangle.vert.wgsl";import fragShaderCode from "./shaders/triangle.frag.wgsl"; // ✋ Declare shader module handleslet vertModule: GPUShaderModule = null;let fragModule: GPUShaderModule = null; const vsmDesc = { code: vertShaderCode };vertModule = device.createShaderModule(vsmDesc); const fsmDesc = { code: fragShaderCode };fragModule = device.createShaderModule(fsmDesc);
```

> In Webpack you can import WebGPU shaders as strings by adding this object to your list of rules:
>
> ```
> Webpack WGSL Loader{  test: /\.(wgsl|glsl)$/,  type: "asset/source",}
> ```

<a id="uniform-buffer"></a>

### Uniform Buffer

![Uniform Buffer Diagram](<raw-webgpu/192abee30435a94454c20c85.svg>)

You'll often times need to feed data directly to your shader modules, and to do this you'll need to specify a uniform. In order to create a **Uniform Buffer** in your shader, declare the following prior to your main function:

```
WGSL Uniformsstruct UBO {  modelViewProj: mat4x4<f32>,  primaryColor: vec4<f32>,  accentColor: vec4<f32>}; @group(0) @binding(0)var<uniform> uniforms: UBO; // ❗ Then in your Vertex Shader's main file,// replace the 4th to last line with:vsOut.Position = uniforms.modelViewProj * vec4<f32>(inPos, 1.0);
```

Then in your JavaScript code, create a uniform buffer as you would with an index/vertex buffer.

> You'll want to use a library like [gl-matrix](https://github.com/toji/gl-matrix) in order to better manage linear algebra calculations such as matrix multiplication.

```
Creating uniform buffers// 👔 Uniform Dataconst uniformData = new Float32Array([     // ♟️ ModelViewProjection Matrix (Identity)    1.0, 0.0, 0.0, 0.0    0.0, 1.0, 0.0, 0.0    0.0, 0.0, 1.0, 0.0    0.0, 0.0, 0.0, 1.0     // 🔴 Primary Color    0.9, 0.1, 0.3, 1.0     // 🟣 Accent Color    0.8, 0.2, 0.8, 1.0]); // ✋ Declare buffer handleslet uniformBuffer: GPUBuffer = null; uniformBuffer = createBuffer(uniformData, GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST);
```

<a id="pipeline-layout"></a>

### Pipeline Layout

![Pipeline Layout Diagram](<raw-webgpu/725bd2c59d880547c67adefc.svg>)

Once you have a uniform, you can create a **Pipeline Layout** to describe where that uniform will be when executing a *Graphics Pipeline*.

You have 2 options here, you can either let WebGPU create your pipeline layout for you, or get it from a pipeline that's already been created:

```
Creating Pipeline Layouts with Implicit Layoutslet bindGroupLayout: GPUBindGroupLayout = null;let uniformBindGroup: GPUBindGroup = null; // 👨‍🔧 Create your graphics pipeline... // 🧙‍♂️ Then get your implicit pipeline layout:bindGroupLayout = pipeline.getBindGroupLayout(0); // 🗄️ Bind Group// ✍ This would be used when *encoding commands*uniformBindGroup = device.createBindGroup({  layout: bindGroupLayout,  entries: [    {      binding: 0,      resource: {        buffer: uniformBuffer,      },    },  ],});
```

Or if you know the layout in advance, you can describe it yourself and use it during pipeline creation:

```
Creating Pipeline Layouts Manually// ✋ Declare handleslet uniformBindGroupLayout: GPUBindGroupLayout = null;let uniformBindGroup: GPUBindGroup = null;let layout: GPUPipelineLayout = null; // 📁 Bind Group LayoutuniformBindGroupLayout = device.createBindGroupLayout({  entries: [    {      binding: 0,      visibility: GPUShaderStage.VERTEX,      buffer: {},    },  ],}); // 🗄️ Bind Group// ✍ This would be used when *encoding commands*uniformBindGroup = device.createBindGroup({  layout: uniformBindGroupLayout,  entries: [    {      binding: 0,      resource: {        buffer: uniformBuffer,      },    },  ],}); // 🗂️ Pipeline Layout// 👩‍🔧 This would be used as a member of a GPUPipelineDescriptor when *creating a pipeline*const pipelineLayoutDesc = { bindGroupLayouts: [uniformBindGroupLayout] };layout = device.createPipelineLayout(pipelineLayoutDesc);
```

When encoding commands, you can use this uniform with `setBindGroup`:

```
// ✍ Later when you're encoding commands:
passEncoder.setBindGroup(0, uniformBindGroup);
```

<a id="graphics-pipeline"></a>

### Graphics Pipeline

![Pipeline Diagram](<raw-webgpu/68508ed4f78ab17a8da11dc5.svg>)

A **Graphics Pipeline** describes all the data that's to be fed into the execution of a raster based graphics pipeline. This includes:

- 🔣 **Input Assembly** - What does each vertex look like? Which attributes are where, and how do they align in memory?
- 🖍️ **Shader Modules** - What shader modules will you be using when executing this graphics pipeline?
- ✏️ **Depth/Stencil State** - Should you perform depth testing? If so, what function should you use to test depth?
- 🍥 **Blend State** - How should colors be blended between the previously written color and current one?
- 🔺 **Rasterization** - How does the rasterizer behave when executing this graphics pipeline? Does it cull faces? Which direction should the face be culled?
- 💾 **Uniform Data** - What kind of uniform data should your shaders expect? In WebGPU this is done by describing a **Pipeline Layout**.

WebGPU features smart defaults for the graphics pipeline state, so most of the time you won't need to even set parts of it:

```
GPURenderPipeline Creation// ✋ Declare pipeline handlelet pipeline: GPURenderPipeline = null; // ⚗️ Graphics Pipeline // 🔣 Input Assemblyconst positionAttribDesc: GPUVertexAttribute = {  shaderLocation: 0, // @location(0)  offset: 0,  format: "float32x3",};const colorAttribDesc: GPUVertexAttribute = {  shaderLocation: 1, // @location(1)  offset: 0,  format: "float32x3",};const positionBufferDesc: GPUVertexBufferLayout = {  attributes: [positionAttribDesc],  arrayStride: 4 * 3, // sizeof(float) * 3  stepMode: "vertex",};const colorBufferDesc: GPUVertexBufferLayout = {  attributes: [colorAttribDesc],  arrayStride: 4 * 3, // sizeof(float) * 3  stepMode: "vertex",}; // 🌑 Depthconst depthStencil: GPUDepthStencilState = {  depthWriteEnabled: true,  depthCompare: "less",  format: "depth24plus-stencil8",}; // 🦄 Uniform Dataconst pipelineLayoutDesc = { bindGroupLayouts: [] };const layout = device.createPipelineLayout(pipelineLayoutDesc); // 🎭 Shader Stagesconst vertex: GPUVertexState = {  module: vertModule,  entryPoint: "main",  buffers: [positionBufferDesc, colorBufferDesc],}; // 🌀 Color/Blend Stateconst colorState: GPUColorTargetState = {  format: "bgra8unorm",}; const fragment: GPUFragmentState = {  module: fragModule,  entryPoint: "main",  targets: [colorState],}; // 🟨 Rasterizationconst primitive: GPUPrimitiveState = {  frontFace: "cw",  cullMode: "none",  topology: "triangle-list",}; const pipelineDesc: GPURenderPipelineDescriptor = {  layout,  vertex,  fragment,  primitive,  depthStencil,}; pipeline = device.createRenderPipeline(pipelineDesc);
```

<a id="command-encoder"></a>

### Command Encoder

![Command Encoder Diagrams](<raw-webgpu/d63d9a31c835ea584f5e6d2b.svg>)

**Command Encoder**s encode all the draw commands you intend to execute in groups of **Render Pass Encoders**. Once you've finished encoding commands, you'll receive a **Command Buffer** that you could submit to your queue.

In that sense a command buffer is analogous to a *callback* that executes draw functions on the GPU once it's submitted to the queue.

```
Encoding Commands// ✋ Declare command handleslet commandEncoder: GPUCommandEncoder = null;let passEncoder: GPURenderPassEncoder = null; // ✍️ Write commands to send to the GPUconst encodeCommands = () => {  let colorAttachment: GPURenderPassColorAttachment = {    view: this.colorTextureView,    clearValue: { r: 0, g: 0, b: 0, a: 1 },    loadOp: "clear",    storeOp: "store",  };   const depthAttachment: GPURenderPassDepthStencilAttachment = {    view: this.depthTextureView,    depthClearValue: 1,    depthLoadOp: "clear",    depthStoreOp: "store",    stencilClearValue: 0,    stencilLoadOp: "clear",    stencilStoreOp: "store",  };   const renderPassDesc: GPURenderPassDescriptor = {    colorAttachments: [colorAttachment],    depthStencilAttachment: depthAttachment,  };   commandEncoder = device.createCommandEncoder();   // 🖌️ Encode drawing commands  passEncoder = commandEncoder.beginRenderPass(renderPassDesc);  passEncoder.setPipeline(pipeline);  passEncoder.setViewport(0, 0, canvas.width, canvas.height, 0, 1);  passEncoder.setScissorRect(0, 0, canvas.width, canvas.height);  passEncoder.setVertexBuffer(0, positionBuffer);  passEncoder.setVertexBuffer(1, colorBuffer);  passEncoder.setIndexBuffer(indexBuffer, "uint16");  passEncoder.drawIndexed(3);  passEncoder.endPass();   queue.submit([commandEncoder.finish()]);};
```

<a id="render"></a>

## Render

![Triangle Raster Gif](<raw-webgpu/4f21a6bcebb42b2089c1b0f1.gif>)

**Rendering** in WebGPU is a simple matter of updating any uniforms you intend to update, getting the next attachments from your context, submitting your command encoders to be executed, and using the `requestAnimationFrame` callback to do all of that again.

```
Basic Render Loopconst render = () => {  // ⏭ Acquire next image from context  colorTexture = context.getCurrentTexture();  colorTextureView = colorTexture.createView();   // 📦 Write and submit commands to queue  encodeCommands();   // ➿ Refresh canvas  requestAnimationFrame(render);};
```

<a id="conclusion"></a>

## Conclusion

WebGPU might be more difficult than other graphics APIs, but it's an API that *more closely aligns with the design of modern graphics cards*, and as a result, should not only result in faster applications, but also applications that should last longer.

There were a few things I didn't cover in this post as they would have been a beyond the scope of this post, such as:

- **Matrices**, be it for cameras or for transforming objects in the scene. [gl-matrix](https://github.com/toji/gl-matrix) is an invaluable resource there.
- A detailed overview of **every possible state of a graphics pipeline**. The [WebGPU type definitions](https://github.com/gpuweb/types) are very helpful there.
- **Blend Modes**, it can be helpful to see this visually, Anders Riggelsen wrote a tool to [see blend mode behavior with OpenGL here](https://www.andersriggelsen.dk/glblendfunc.php).
- **Compute pipelines**, review the specification or some of the examples below if you want to try that.
- **Loading textures**, this can be a bit involved, the examples below introduce this very well.

<a id="additional-resources"></a>

### Additional Resources

Here's a few articles/projects for WebGPU in *no particular order*:

- [WebGPU Fundimentals](https://webgpufundamentals.org/) is an online public book [written and edited by various authors on GitHub](https://github.com/webgpu/webgpufundamentals) that tries to be both comprehensive and consise.
- ![William Usher's picture](<raw-webgpu/b612874a8bb3327e54664a6e.jpeg>) William Usher ([@\_wusher](https://twitter.com/_wusher))'s article, updated
  for the Chrome 113 release of WebGPU: [From 0 to glTF with WebGPU](https://www.willusher.io/graphics/2023/04/10/0-to-gltf-triangle).
- [Dzmitry Malyshau](https://github.com/kvark) wrote an article similar to this one [introducing WebGPU in Mozilla FireFox](https://hacks.mozilla.org/2020/04/experimental-webgpu-in-firefox/).
- [Warren Moore](https://metalbyexample.com/) ([@warrenm](https://twitter.com/warrenm)) wrote an article to help folks transition from the Metal API to [WebGPU](https://metalbyexample.com/webgpu-part-one/).
- [Brandon Jones](https://blog.tojicode.com/) ([@Tojiro](https://twitter.com/Tojiro)) wrote an article describing how to [write a GLTF renderer in WebGPU](https://toji.github.io/webgpu-gltf-case-study/).
- [Learn WGPU](https://sotrh.github.io/learn-wgpu/#what-is-wgpu) is an introduction to writing WebGPU applications with Rust.
- [Learn WebGPU](https://eliemichel.github.io/LearnWebGPU/) for native graphics in C++.

There's also a number of open source projects including:

- ![Austin Eng picture](<raw-webgpu/12f354c3e42073f7bc4d17f0.png>) [Austin Eng](http://austineng.github.io/)'s [WebGPU Samples](http://austin-eng.com/webgpu-samples/samples/texturedCube)
- ![Tarek Sherif picture](<raw-webgpu/6ff2c45285478e0e7a22f928.jpeg>) [Tarek Sherif](http://tareksherif.net/) ([@tsherif](https://twitter.com/thsherif))'s
  [WebGPU Examples](https://github.com/tsherif/webgpu-examples)
- [RedGPU](https://github.com/redcamel/RedGPU) by [@RedCamel15](https://twitter.com/redcamel15), a series of examples written for WebGPU.
- [Three.js' WebGPU Source](https://github.com/mrdoob/three.js/tree/dev/examples/jsm/renderers/webgpu)
- [BabylonJS's WebGPU Source](https://github.com/BabylonJS/Babylon.js/blob/WebGPU/src/Engines/webgpuEngine.ts)
- [WebGPU's Type Definitions](https://github.com/gpuweb/types)
- [WebGPU's Conformance Tests](https://github.com/gpuweb/cts)
- [Dawn - A C++ Implementation of WebGPU](https://dawn.googlesource.com/dawn) used to power Chromium's implementation of WebGPU. Carl Woffenden released a [Hello Triangle example with WebGPU and Dawn](https://github.com/cwoffenden/hello-webgpu).
- [wgpu-native](https://github.com/gfx-rs/wgpu-native) is a native WebGPU implementation in Rust, which can easily be used in all sorts of native projects.

The specification for WebGPU and WebGPU Shading Language is also worth taking a look:

- [WebGPU Specification](https://gpuweb.github.io/gpuweb/)
- [WebGPU Shading Language (WGSL) Specification](https://gpuweb.github.io/gpuweb/wgsl/)

---

You can find all the source for this post in the [GitHub Repo](https://github.com/alaingalvan/webgpu-seed) here.

Source: [https://alain.xyz/blog/raw-webgpu](<https://alain.xyz/blog/raw-webgpu>)
