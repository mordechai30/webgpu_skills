# Your first WebGPU app

## [3. Initialize WebGPU](https://codelabs.developers.google.com/your-first-webgpu-app#2)

<a id="start-with-a-canvas"></a>

## Start with a `<canvas>`

WebGPU can be used without showing anything on the screen if all you want is to use it to do computations. But if you want to render anything, like we're going to be doing in the codelab, you need a canvas. So that's a good spot to start!

Create a new HTML document with a single `<canvas>` element in it, as well as a `<script>` tag where we query the canvas element. (Or use [00-starter-page.html](https://github.com/GoogleChromeLabs/your-first-webgpu-app-codelab/blob/main/00-starter-page.html).)

- Create an `index.html` file with the following code:

<a id="index.html"></a>

### **index.html**

```
<!doctype html>

<html>
  <head>
    <meta charset="utf-8">
    <title>WebGPU Life</title>
  </head>
  <body>
    <canvas width="512" height="512"></canvas>
    <script type="module">
      const canvas = document.querySelector("canvas");

      // Your WebGPU code will begin here!
    </script>
  </body>
</html>
```



**Note:** Canvas elements have a default size of 300x150 pixels, which is small for this app's needs. Although it's not required to use WebGPU, you probably want to give the canvas a larger width and height. The example code above makes the canvas 512x512 CSS pixels, but you're free to choose whatever size you want.



**Note:** Giving the script tag a type of `"module"` allows you to use [top-level awaits](https://v8.dev/features/top-level-await), which is useful for WebGPU initialization!



<a id="request-an-adapter-and-device"></a>

## **Request an adapter and device**

Now you can get into the WebGPU bits! First, you should consider that APIs like WebGPU can take a while to propagate throughout the entire web ecosystem. As a result, a good first precautionary step is to check if the user's browser can use WebGPU.

1. To check if the [`navigator.gpu`](https://gpuweb.github.io/gpuweb/#navigator-gpu) object, which serves as the entry point for WebGPU, exists, add the following code:

<a id="index.html_1"></a>

### **index.html**

```
if (!navigator.gpu) {
  throw new Error("WebGPU not supported on this browser.");
}
```

Ideally, you want to inform the user if WebGPU is unavailable by having the page fall back to a mode that doesn't use WebGPU. (Maybe it could use WebGL instead?) For the purposes of this codelab, though, you just throw an error to stop the code from executing further.

Once you know that WebGPU is supported by the browser, the first step in initializing WebGPU for your app is to request a [`GPUAdapter`](https://gpuweb.github.io/gpuweb/#gpuadapter). You can think of an adapter as WebGPU's representation of a specific piece of GPU hardware in your device.

2. To get an adapter, use the [`navigator.gpu.requestAdapter()`](https://gpuweb.github.io/gpuweb/#dom-gpu-requestadapter) method. It returns a promise, so it's most convenient to call it with `await`.

<a id="index.html_2"></a>

### **index.html**

```
const adapter = await navigator.gpu.requestAdapter();
if (!adapter) {
  throw new Error("No appropriate GPUAdapter found.");
}
```

If no appropriate adapters can be found, the returned `adapter` value might be `null`, so you want to handle that possibility. It might happen if the user's browser supports WebGPU but their GPU hardware doesn't have all the features necessary to use WebGPU.

Most of the time it's OK to simply let the browser pick a default adapter, as you do here, but for more advanced needs there are [arguments that can be passed](https://gpuweb.github.io/gpuweb/#adapter-selection) to `requestAdapter()` that specify whether you want to use low-power or high-performance hardware on devices with multiple GPUs (like some laptops).

Once you have an adapter, the last step before you can start working with the GPU is to request a [GPUDevice](https://gpuweb.github.io/gpuweb/#gpudevice). The device is the main interface through which most interaction with the GPU happens.

3. Get the device by calling [`adapter.requestDevice()`](https://gpuweb.github.io/gpuweb/#dom-gpuadapter-requestdevice), which also returns a promise.

<a id="index.html_3"></a>

### **index.html**

```
const device = await adapter.requestDevice();
```

As with `requestAdapter()`, there are [options that can be passed](https://gpuweb.github.io/gpuweb/#gpudevicedescriptor) here for more advanced uses like enabling specific hardware features or requesting higher limits, but for your purposes the defaults work just fine.

<a id="configure-the-canvas"></a>

## **Configure the Canvas**

Now that you have a device, there's one more thing to do if you want to use it to show anything on the page: configure the canvas to be used with the device you just created.

- To do this, first request a [`GPUCanvasContext`](https://gpuweb.github.io/gpuweb/#canvas-context) from the canvas by calling `canvas.getContext("webgpu")`. (This is the same call that you'd use to initialize Canvas 2D or WebGL contexts, using the `2d` and `webgl` context types, respectively.) The `context` that it returns must then be associated with the device using the [`configure()`](https://gpuweb.github.io/gpuweb/#dom-gpucanvascontext-configure) method, like so:

<a id="index.html_4"></a>

### **index.html**

```
const context = canvas.getContext("webgpu");
const canvasFormat = navigator.gpu.getPreferredCanvasFormat();
context.configure({
  device: device,
  format: canvasFormat,
});
```

There are a few options that can be passed here, but the most important ones are the *`device`* that you are going to use the context with and the *`format`*, which is the *texture format* that the context should use.

Textures are the objects that WebGPU uses to store image data, and each texture has a format that lets the GPU know how that data is laid out in memory. The details of how texture memory works are beyond the scope of this codelab. The important thing to know is that the canvas context provides textures for your code to draw into, and the format that you use can have an impact on how efficiently the canvas shows those images. Different types of devices perform best when using different texture formats, and if you don't use the device's preferred format it may cause extra memory copies to happen behind the scenes before the image can be displayed as part of the page.

Fortunately, you don't have to worry much about any of that because WebGPU tells you which format to use for your canvas! In almost all cases, you want to pass the value returned by calling [`navigator.gpu.getPreferredCanvasFormat()`](https://gpuweb.github.io/gpuweb/#dom-gpu-getpreferredcanvasformat), as shown above.

**Note:** One big difference in how WebGPU works compared to WebGL is that because canvas configuration is separate from device creation you can have any number of canvases that are all being rendered by a single device! This will make certain use cases, like multi-pane 3D editors, much easier to develop.



<a id="clear-the-canvas"></a>

## **Clear the Canvas**

Now that you have a device and the canvas has been configured with it, you can start using the device to change the content of the canvas. To start, clear it with a solid color.

In order to do that—or pretty much anything else in WebGPU—you need to provide some commands to the GPU instructing it what to do.

1. To do this, have the device create a [`GPUCommandEncoder`](https://gpuweb.github.io/gpuweb/#gpucommandencoder), which provides an interface for recording GPU commands.

<a id="index.html_5"></a>

### **index.html**

```
const encoder = device.createCommandEncoder();
```

The commands you want to send to the GPU are related to rendering (in this case, clearing the canvas), so the next step is to use the `encoder` to begin a Render Pass.

Render passes are when all drawing operations in WebGPU happen. Each one starts off with a [`beginRenderPass()`](https://gpuweb.github.io/gpuweb/#dom-gpucommandencoder-beginrenderpass) call, which defines the textures that receive the output of any drawing commands performed. More advanced uses can provide several textures, called *attachments*, with various purposes such as storing the depth of rendered geometry or providing antialiasing. For this app, however, you only need one.

2. Get the texture from the canvas context you created earlier by calling [`context.getCurrentTexture()`](https://gpuweb.github.io/gpuweb/#dom-gpucanvascontext-getcurrenttexture), which returns a texture with a pixel width and height matching the canvas' `width` and `height` attributes and the `format` specified when you called `context.configure()`.

<a id="index.html_6"></a>

### **index.html**

```
const pass = encoder.beginRenderPass({
  colorAttachments: [{
     view: context.getCurrentTexture().createView(),
     loadOp: "clear",
     storeOp: "store",
  }]
});
```

The texture is given as the `view` property of a `colorAttachment`**.** Render passes require that you provide a `GPUTextureView` instead of a `GPUTexture`, which tells it which parts of the texture to render to. This only really matters for more advanced use cases, so here you call `createView()` with no arguments on the texture, indicating that you want the render pass to use the entire texture.

You also have to specify what you want the render pass to do with the texture when it starts and when it ends:

- A `loadOp` value of `"clear"` indicates that you want the texture to be cleared when the render pass starts.
- A `storeOp` value of `"store"` indicates that once the render pass is finished you want the results of any drawing done during the render pass saved into the texture.

Once the render pass has begun you do... nothing! At least for now. The act of starting the render pass with `loadOp: "clear"` is enough to clear the texture view and the canvas.

3. End the render pass by adding the following call immediately after `beginRenderPass()`:

<a id="index.html_7"></a>

### **index.html**

```
pass.end();
```

It's important to know that simply making these calls does not cause the GPU to actually do anything. They're just recording commands for the GPU to do later.

4. In order to create a `GPUCommandBuffer`, call `finish()` on the command encoder. The command buffer is an opaque handle to the recorded commands.

<a id="index.html_8"></a>

### **index.html**

```
const commandBuffer = encoder.finish();
```

5. Submit the command buffer to the GPU using the `queue` of the `GPUDevice`. The queue performs all GPU commands, ensuring that their execution is well ordered and properly synchronized. The queue's `submit()` method takes in an array of command buffers, though in this case you only have one.

<a id="index.html_9"></a>

### **index.html**

```
device.queue.submit([commandBuffer]);
```

Once you submit a command buffer, it cannot be used again, so there's no need to hold on to it. If you want to submit more commands, you need to build another command buffer. That's why it's fairly common to see those two steps collapsed into one, as is done in the sample pages for this codelab:

<a id="index.html_10"></a>

### **index.html**

```
// Finish the command buffer and immediately submit it.
device.queue.submit([encoder.finish()]);
```

After you submit the commands to the GPU, let JavaScript return control to the browser. At that point, the browser sees that you've changed the current texture of the context and updates the canvas to display that texture as an image. If you want to update the canvas contents again after that, you need to record and submit a new command buffer, calling `context.getCurrentTexture()` again to get a new texture for a render pass.

6. Reload the page. Notice that the canvas is filled with black. Congratulations! That means that you've successfully created your first WebGPU app.

![A black canvas that indicates that WebGPU has successfully been used to clear the canvas contents. ](<your-first-webgpu-app/9acea46aa0ffb468fa9ba145.png>)

<a id="pick-a-color"></a>

## **Pick a color!**

To be honest, though, black squares are pretty boring. So take a moment before moving on to the next section in order to personalize it just a bit.

1. In the `encoder.beginRenderPass()` call, add a new line with a `clearValue` to the `colorAttachment`, like this:

<a id="index.html_11"></a>

### **index.html**

```
const pass = encoder.beginRenderPass({
  colorAttachments: [{
    view: context.getCurrentTexture().createView(),
    loadOp: "clear",
    clearValue: { r: 0, g: 0, b: 0.4, a: 1 }, // New line
    storeOp: "store",
  }],
});
```

The `clearValue` instructs the render pass which color it should use when performing the `clear` operation at the beginning of the pass. The dictionary passed into it contains four values: `r` for *red*, `g` for *green*, `b` for *blue*, and `a` for *alpha* (transparency). Each value can range from `0` to `1`, and together they describe the value of that color channel. For example:

- `{ r: 1, g: 0, b: 0, a: 1 }` is bright red.
- `{ r: 1, g: 0, b: 1, a: 1 }` is bright purple.
- `{ r: 0, g: 0.3, b: 0, a: 1 }` is dark green.
- `{ r: 0.5, g: 0.5, b: 0.5, a: 1 }` is medium gray.
- `{ r: 0, g: 0, b: 0, a: 0 }` is the default, transparent black.

**Note:** Defining colors in WebGPU is very similar to setting CSS colors with the [rgb() functional notation](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/rgb).



**Pro Tip:** For a little bit less typing, you can also use an array shorthand, giving values in RGBA order. Passing `[0, 0.5, 0.7, 1]` is the same as passing `{ r: 0, g: 0.5, b: 0.7, a: 1 }`.

The example code and screenshots in this codelab use a dark blue, but feel free to pick whatever color you want!

2. Once you've picked your color, reload the page. You should see your chosen color in the canvas.

![A canvas cleared to a dark blue color to demonstrate how to change the default clear color.](<your-first-webgpu-app/56f3a52482d93a747abe7ef5.png>)

<a id="4-draw-geometry"></a>

Source: [https://codelabs.developers.google.com/your-first-webgpu-app#2](<https://codelabs.developers.google.com/your-first-webgpu-app#2>)
