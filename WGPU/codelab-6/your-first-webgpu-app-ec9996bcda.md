# Your first WebGPU app

## [7. Manage cell state](https://codelabs.developers.google.com/your-first-webgpu-app#6)

Next, you need to control which cells on the grid render, based on some state that's stored on the GPU. This is important for the final simulation!

All you need is an on-off signal for each cell, so any options that allow you to store a large array of nearly any value type works. You might think that this is another use case for uniform buffers! While you *could* make that work, it's more difficult because uniform buffers are limited in size, can't support dynamically sized arrays (you have to specify the array size in the shader), and can't be written to by compute shaders. That last item is the most problematic, since you want to do the Game of Life simulation on the GPU in a compute shader.

Fortunately, there's another buffer option that avoids all of those limitations.

<a id="create-a-storage-buffer"></a>

## Create a storage buffer

Storage buffers are general-use buffers that can be read and written to in compute shaders, and read in vertex shaders. They can be *very* large, and they don't need a specific declared size in a shader, which makes them much more like general memory. That's what you use to store the cell state.

**Note:** If storage buffers are so much more flexible, why bother with uniform buffers at all? It actually depends on your GPU hardware! There's a good chance that uniform buffers are given special treatment by your GPU in order to allow them to update and be read faster than a storage buffer, so for smaller amounts of data that have the potential to update frequently (like model, view, and projection matrices in 3D applications), uniforms are typically the safer choice for better performance.

1. To create a storage buffer for your cell state, use what—by now—is probably starting to be a familiar-looking snippet of buffer creation code:

**index.html**

```
// Create an array representing the active state of each cell.
const cellStateArray = new Uint32Array(GRID_SIZE * GRID_SIZE);

// Create a storage buffer to hold the cell state.
const cellStateStorage = device.createBuffer({
  label: "Cell State",
  size: cellStateArray.byteLength,
  usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
});
```

Just like with your vertex and uniform buffers, call `device.createBuffer()` with the appropriate size, and then make sure to specify a usage of `GPUBufferUsage.STORAGE` this time.

You can populate the buffer the same way as before by filling the TypedArray of the same size with values and then calling `device.queue.writeBuffer()`. Because you want to see the effect of your buffer on the grid, start by filling it with something predictable.

2. Activate every third cell with the following code:

**index.html**

```
// Mark every third cell of the grid as active.
for (let i = 0; i < cellStateArray.length; i += 3) {
  cellStateArray[i] = 1;
}
device.queue.writeBuffer(cellStateStorage, 0, cellStateArray);
```



<a id="read-the-storage-buffer-in-the-shader"></a>

## **Read the storage buffer in the shader**

Next, update your shader to look at the contents of the storage buffer before you render the grid. This looks very similar to how uniforms were added previously.

1. Update your shader with the following code:

**index.html**

```
@group(0) @binding(0) var<uniform> grid: vec2f;
@group(0) @binding(1) var<storage> cellState: array<u32>; // New!
```

First, you add the binding point, which tucks right underneath the grid uniform. You want to keep the same `@group` as the `grid` uniform, but the `@binding` number needs to be different. The `var` type is `storage`, in order to reflect the different type of buffer, and rather than a single vector, the type that you give for the `cellState` is an array of `u32` values, in order to match the `Uint32Array` in JavaScript.

**Note:** Why 32 bit ints? Doesn't that seem wasteful if all you need are booleans? Well, yeah! It is! But also, the GPU only cleanly exposes data of a few types because it can work with them fast. You can't specify that you're exposing an array of bytes, for example, because it wouldn't work well with certain assumptions GPUs make about data alignment.

Now you *could* save space and use bitmasking tricks such as storing the active state for 32 different cells in each value of that array. There's actually a long history of developers finding clever ways to pack the data they need into GPU-approved types like that! But that would make the example code far more complex. And the truth is that for a use case as relatively simple as this one, you don't need to worry about the memory impact too much.

Next, in the body of your `@vertex` function, query the cell's state. Because the state is stored in a flat array in the storage buffer, you can use the `instance_index` in order to look up the value for the current cell!

How do you turn off a cell if the state says that it's inactive? Well, since the active and inactive states that you get from the array are 1 or 0, you can scale the geometry by the active state! Scaling it by 1 leaves the geometry alone, and scaling it by 0 makes the geometry collapse into a single point, which the GPU then discards.

2. Update your shader code to scale the position by the cell's active state. The state value must be cast to a `f32` in order to satisfy WGSL's type safety requirements:

**index.html**

```
@vertex
fn vertexMain(@location(0) pos: vec2f,
              @builtin(instance_index) instance: u32) -> VertexOutput {
  let i = f32(instance);
  let cell = vec2f(i % grid.x, floor(i / grid.x));
  let state = f32(cellState[instance]); // New line!

  let cellOffset = cell / grid * 2;
  // New: Scale the position by the cell's active state.
  let gridPos = (pos*state+1) / grid - 1 + cellOffset;

  var output: VertexOutput;
  output.pos = vec4f(gridPos, 0, 1);
  output.cell = cell;
  return output;
}
```



<a id="add-the-storage-buffer-to-the-bind-group"></a>

## **Add the storage buffer to the bind group**

Before you can see the cell state take effect, add the storage buffer to a bind group. Because it's part of the same `@group` as the uniform buffer, add it to the same bind group in the JavaScript code, as well.

- Add the storage buffer, like this:

**index.html**

```
const bindGroup = device.createBindGroup({
  label: "Cell renderer bind group",
  layout: cellPipeline.getBindGroupLayout(0),
  entries: [{
    binding: 0,
    resource: { buffer: uniformBuffer }
  },
  // New entry!
  {
    binding: 1,
    resource: { buffer: cellStateStorage }
  }],
});
```

Make sure that the `binding` of the new entry matches the `@binding()` of the corresponding value in the shader!

With that in place, you should be able to refresh and see the pattern appear in the grid.

![Diagonal stripes of colorful squares going from bottom left to top right against a dark blue background. ](<your-first-webgpu-app/1f5218012d24c59a53bd1f0c.png>)

<a id="use-the-ping-pong-buffer-pattern"></a>

## **Use the ping-pong buffer pattern**

Most simulations like the one you're building typically use at least *two* copies of their state. On each step of the simulation, they read from one copy of the state and write to the other. Then, on the next step, flip it and read from the state they wrote to previously. This is commonly referred to as a [ping pong](https://en.wikipedia.org/wiki/Ping-pong_scheme) pattern because the most up-to-date version of the state bounces back and forth between state copies each step.

Why is that necessary? Look at a simplified example: imagine that you're writing a very simple simulation in which you move any active blocks right by one cell each step. To keep things easy to understand, you define your data and simulation in JavaScript:

```
// Example simulation. Don't copy into the project!
const state = [1, 0, 0, 0, 0, 0, 0, 0];

function simulate() {
  for (let i = 0; i < state.length-1; ++i) {
    if (state[i] == 1) {
      state[i] = 0;
      state[i+1] = 1;
    }
  }
}

simulate(); // Run the simulation for one step.
```

But if you run that code, the active cell moves all the way to the end of the array in one step! Why? Because you keep updating the state in-place, so you move the active cell right, and then you look at the next cell and... hey! It's active! Better move it to the right again. The fact that you change the data at the same time that you observe it corrupts the results.

By using the ping pong pattern, you ensure that you always perform the next step of the simulation using *only* the results of the last step.

```
// Example simulation. Don't copy into the project!
const stateA = [1, 0, 0, 0, 0, 0, 0, 0];
const stateB = [0, 0, 0, 0, 0, 0, 0, 0];

function simulate(inArray, outArray) {
  outArray[0] = 0;
  for (let i = 1; i < inArray.length; ++i) {
     outArray[i] = inArray[i-1];
  }
}

// Run the simulation for two step.
simulate(stateA, stateB);
simulate(stateB, stateA); 
```

1. Use this pattern in your own code by updating your storage buffer allocation in order to create two identical buffers:

**index.html**

```
// Create two storage buffers to hold the cell state.
const cellStateStorage = [
  device.createBuffer({
    label: "Cell State A",
    size: cellStateArray.byteLength,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
  }),
  device.createBuffer({
    label: "Cell State B",
     size: cellStateArray.byteLength,
     usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
  })
];
```

2. To help visualize the difference between the two buffers, fill them with different data:

**index.html**

```
// Mark every third cell of the first grid as active.
for (let i = 0; i < cellStateArray.length; i+=3) {
  cellStateArray[i] = 1;
}
device.queue.writeBuffer(cellStateStorage[0], 0, cellStateArray);

// Mark every other cell of the second grid as active.
for (let i = 0; i < cellStateArray.length; i++) {
  cellStateArray[i] = i % 2;
}
device.queue.writeBuffer(cellStateStorage[1], 0, cellStateArray);
```



**Note:** As the above code snippet shows, once you call `writeBuffer()`, you don't have to preserve the contents of the source TypedArray any more. At that point, the contents have been copied and the GPU buffer is guaranteed to receive the data as it was at the time the call is made. This allows you to reuse the JavaScript object for the next upload, which saves on memory!

3. To show the different storage buffers in your rendering, update your bind groups to have two different variants, as well:

**index.html**

```
const bindGroups = [
  device.createBindGroup({
    label: "Cell renderer bind group A",
    layout: cellPipeline.getBindGroupLayout(0),
    entries: [{
      binding: 0,
      resource: { buffer: uniformBuffer }
    }, {
      binding: 1,
      resource: { buffer: cellStateStorage[0] }
    }],
  }),
   device.createBindGroup({
    label: "Cell renderer bind group B",
    layout: cellPipeline.getBindGroupLayout(0),
    entries: [{
      binding: 0,
      resource: { buffer: uniformBuffer }
    }, {
      binding: 1,
      resource: { buffer: cellStateStorage[1] }
    }],
  })
];
```



<a id="set-up-a-render-loop"></a>

## **Set up a render loop**

So far, you've only done one draw per page refresh, but now you want to show data updating over time. To do that you need a simple render loop.

A render loop is an endlessly repeating loop that draws your content to the canvas at a certain interval. Many games and other content that want to animate smoothly use the `requestAnimationFrame()` function to schedule callbacks at the same rate that the screen refreshes (60 times every second).

This app can use that, as well, but in this case, you probably want updates to happen in longer steps so that you can more easily follow what the simulation is doing. Manage the loop yourself instead so that you can control the rate at which your simulation updates.

1. First, pick a rate for our simulation to update at (200ms is good, but you can go slower or faster if you like), and then keep track of how many steps of simulation have been completed.

**index.html**

```
const UPDATE_INTERVAL = 200; // Update every 200ms (5 times/sec)
let step = 0; // Track how many simulation steps have been run
```

2. Then move all of the code you currently use for rendering into a new function. Schedule that function to repeat at your desired interval with `setInterval()`. Make sure that the function also updates the step count, and use that to pick which of the two bind groups to bind.

**index.html**

```
// Move all of our rendering code into a function
function updateGrid() {
  step++; // Increment the step count
  
  // Start a render pass 
  const encoder = device.createCommandEncoder();
  const pass = encoder.beginRenderPass({
    colorAttachments: [{
      view: context.getCurrentTexture().createView(),
      loadOp: "clear",
      clearValue: { r: 0, g: 0, b: 0.4, a: 1.0 },
      storeOp: "store",
    }]
  });

  // Draw the grid.
  pass.setPipeline(cellPipeline);
  pass.setBindGroup(0, bindGroups[step % 2]); // Updated!
  pass.setVertexBuffer(0, vertexBuffer);
  pass.draw(vertices.length / 2, GRID_SIZE * GRID_SIZE);

  // End the render pass and submit the command buffer
  pass.end();
  device.queue.submit([encoder.finish()]);
}

// Schedule updateGrid() to run repeatedly
setInterval(updateGrid, UPDATE_INTERVAL);
```

And now when you run the app you see that the canvas flips back and forth between showing the two state buffers you created.

![Diagonal stripes of colorful squares going from bottom left to top right against a dark blue background. ](<your-first-webgpu-app/1f5218012d24c59a53bd1f0c.png>) ![Vertical stripes of colorful squares against a dark blue background. ](<your-first-webgpu-app/7c86c00002f0c6e0177f971a.png>)

With that, you're pretty much done with the rendering side of things! You're all set to display the output of the Game of Life simulation you build in the next step, where you finally start using compute shaders.

Obviously there is so much more to WebGPU's rendering capabilities than the tiny slice that you explored here, but the rest is beyond the scope of this codelab. Hopefully, it gives you enough of a taste of how WebGPU's rendering works, though, that it helps make exploring more advanced techniques like 3D rendering easier to grasp.

<a id="8-run-the-simulation"></a>

Source: [https://codelabs.developers.google.com/your-first-webgpu-app#6](<https://codelabs.developers.google.com/your-first-webgpu-app#6>)
