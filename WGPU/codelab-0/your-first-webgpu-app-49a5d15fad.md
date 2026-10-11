# Your first WebGPU app

## [1. Introduction](https://codelabs.developers.google.com/your-first-webgpu-app#0)

![The WebGPU Logo consists of several blue triangles that form a stylized 'W'](<your-first-webgpu-app/8de0fef231bdc8c0a1ed242b.png>)

<a id="what-is-webgpu"></a>

## **What is WebGPU?**

[WebGPU](https://gpuweb.github.io/gpuweb/) is a new, modern API for accessing the capabilities of your GPU in web apps.

<a id="modern-api"></a>

### **Modern API**

Before WebGPU, there was [WebGL](https://registry.khronos.org/webgl/specs/latest/1.0/), which offered a subset of the features of WebGPU. It enabled a new class of rich web content, and developers have built amazing things with it. However, it was based on the [OpenGL ES 2.0](https://www.khronos.org/opengles/) API, released in 2007, which was based on the even older OpenGL API. GPUs have evolved significantly in that time, and the native APIs that are used to interface with them have evolved as well with [Direct3D 12](https://learn.microsoft.com/en-us/windows/win32/direct3d12/what-is-directx-12-), [Metal](https://developer.apple.com/metal/), and [Vulkan](https://www.vulkan.org/).

WebGPU brings the advancements of these modern APIs to the web platform. It focuses on enabling GPU features in a cross-platform way, while presenting an API that feels natural on the web and is less verbose than some of the native APIs it's built on top of.

<a id="rendering"></a>

### **Rendering**

GPUs are often associated with rendering fast, detailed graphics, and WebGPU is no exception. It has the features required to support many of today's most popular rendering techniques across both desktop and mobile GPUs, and provides a path for new features to be added in the future as hardware capabilities continue to evolve.

<a id="compute"></a>

### **Compute**

In addition to rendering, WebGPU unlocks the potential of your GPU for performing general purpose, highly parallel workloads. These *compute shaders* can be used standalone, without any rendering component, or as a tightly integrated part of your rendering pipeline.

In today's codelab you'll learn how to take advantage of both the rendering and compute capabilities of WebGPU in order to create a simple introductory project!

<a id="what-youll-build"></a>

## **What you'll build**

In this codelab, you build [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) using WebGPU. Your app will:

- Use WebGPU's rendering capabilities to draw simple 2D graphics.
- Use WebGPU's compute capabilities to perform the simulation.

![A screenshot of the final product of this codelab](<your-first-webgpu-app/25daed54056cbbf89b927104.png>)

**Note:** To simplify this codelab, all code snippets are vanilla JavaScript. WebGPU works perfectly well with [TypeScript](https://www.typescriptlang.org/), though you'd need to include the [@webgpu/types](https://github.com/gpuweb/types) definitions. Once the API is more widely implemented, the types should become part of the standard TypeScript DOM and worker libraries.

The Game of Life is what's known as a cellular automaton, in which a grid of *cells* change state over time based on some set of rules. In the Game of Life cells become active or inactive depending on how many of their neighboring cells are active, which leads to interesting patterns that fluctuate as you watch.

<a id="what-youll-learn"></a>

## **What you'll learn**

- How to set up WebGPU and configure a canvas.
- How to draw simple 2D geometry.
- How to use vertex and fragment shaders in order to modify what's being drawn.
- How to use compute shaders in order to perform a simple simulation.

This codelab focuses on introducing the fundamental concepts behind WebGPU. It's not intended to be a comprehensive review of the API, nor does it cover (or require) frequently related topics such as 3D matrix math.

<a id="what-youll-need"></a>

## **What you'll need**

- A recent version of Chrome (113 or later) on ChromeOS, macOS, or Windows. WebGPU is a cross-browser, cross-platform API but has not yet shipped everywhere.
- Knowledge of HTML, JavaScript, and [Chrome DevTools](https://developer.chrome.com/devtools).

Familiarity with other Graphics APIs, such as WebGL, Metal, Vulkan, or Direct3D, is **not required**, but if you have any experience with them you'll likely notice lots of similarities with WebGPU that may help jump-start your learning!

<a id="2-get-set-up"></a>

Source: [https://codelabs.developers.google.com/your-first-webgpu-app#0](<https://codelabs.developers.google.com/your-first-webgpu-app#0>)
