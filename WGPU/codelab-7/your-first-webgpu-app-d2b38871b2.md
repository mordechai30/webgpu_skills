# Your first WebGPU app

## [8. Run the simulation](https://codelabs.developers.google.com/your-first-webgpu-app#7)

Now, for the last major piece of the puzzle: performing the Game of Life simulation in a compute shader!

<a id="use-compute-shaders,-at-last"></a>

## **Use compute shaders, at last!**

You've learned abstractly about compute shaders throughout this codelab, but what exactly are they?

A compute shader is similar to vertex and fragment shaders in that they are designed to run with extreme parallelism on the GPU, but unlike the other two shader stages, they don't have a specific set of inputs and outputs. You are reading and writing data exclusively from sources you choose, like storage buffers. This means that instead of executing once for each vertex, instance, or pixel, you have to tell it how many invocations of the shader function you want. Then, when you run the shader, you are told which invocation is being processed, and you can decide what data you are going to access and which operations you are going to perform from there.

Compute shaders must be created in a shader module, just like vertex and fragment shaders, so add that to your code to get started. As you might guess, given the structure of the other shaders that you've implemented, the main function for your compute shader needs to be marked with the `@compute` attribute.

1. Create a compute shader with the following code:

**index.html**

```
// Create the compute shader that will process the simulation.
const simulationShaderModule = device.createShaderModule({
  label: "Game of Life simulation shader",
  code: `
    @compute
    fn computeMain() {

    }`
});
```

Because GPUs are used frequently for 3D graphics, compute shaders are structured such that you can request that the shader be invoked a specific number of times along an X, Y, and Z axis. This lets you very easily dispatch work that conforms to a 2D or 3D grid, which is great for your use case! You want to call this shader `GRID_SIZE` times `GRID_SIZE` times, once for each cell of your simulation.

Due to the nature of GPU hardware architecture, this grid is divided into *workgroups*. A workgroup has an X, Y, and Z size, and although the sizes can be 1 each, there are often performance benefits to making your workgroups a bit bigger. For your shader, choose a somewhat arbitrary workgroup size of 8 times 8. This is useful to keep track of in your JavaScript code.

2. Define a constant for your workgroup size, like this:

**index.html**

```
const WORKGROUP_SIZE = 8;
```

You also need to add the workgroup size to the shader function itself, which you do using [JavaScript's template literals](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Template_literals) so that you can easily use the constant you just defined.

3. Add the workgroup size to the shader function, like this:

**index.html (Compute createShaderModule call)**

```
@compute
@workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE}) // New line
fn computeMain() {

}
```

This tells the shader that work done with this function is done in (8 x 8 x 1) groups. (Any axis you leave off defaults to 1, although you have to at least specify the X axis.)

**Note:** For more advanced uses of compute shaders, the workgroup size becomes more important. Shader invocations within a single workgroup are allowed to share faster memory and use certain types of synchronization primitives. You don't need any of that, though, since your shader executions are fully independent.

You could make the workgroup size (1 x 1 x 1), and it would still work correctly, but that also restricts how well the GPU can run the shader in parallel. Picking something bigger helps the GPU divide the work better.

There is a theoretical ideal workgroup size for every GPU, but it's dependent on architectural details that WebGPU doesn't expose, so usually you want to pick a number driven by the requirements of the shader. Lacking that, given the wide range of hardware that WebGPU content may run on, 64 is a good number that's unlikely to exceed any hardware limits but still handles large enough batches to be reasonably efficient. (8 x 8 == 64, so your workgroup size follows this advice.)

As with the other shader stages, there's a variety of `@builtin` values that you can accept as input into your compute shader function in order to tell you which invocation you're on and decide what work you need to do.

4. Add a `@builtin` value, like this:

**index.html (Compute createShaderModule call)**

```
@compute @workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE})
fn computeMain(@builtin(global_invocation_id) cell: vec3u) {

}
```

You pass in the `global_invocation_id` builtin, which is a three-dimensional vector of unsigned integers that tells you where in the grid of shader invocations you are. You run this shader once for each cell in your grid. You get numbers like `(0, 0, 0)`, `(1, 0, 0)`, `(1, 1, 0)`... all the way to `(31, 31, 0)`, which means that you can treat it as the cell index you're going to operate on!

Compute shaders can also use uniforms, which you use just like in the vertex and fragment shaders.

5. Use a uniform with your compute shader to tell you the grid size, like this:

**index.html (Compute createShaderModule call)**

```
@group(0) @binding(0) var<uniform> grid: vec2f; // New line

@compute @workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE})
fn computeMain(@builtin(global_invocation_id) cell: vec3u) {

}
```

Just like in the vertex shader, you also expose the cell state as a storage buffer. But in this case, you need two of them! Because compute shaders don't have a required output, like a vertex position or fragment color, writing values to a storage buffer or texture is the only way to get results out of a compute shader. Use the ping-pong method that you learned earlier; you have one storage buffer that feeds in the current state of the grid and one that you write out the new state of the grid to.

6. Expose the cell input and output state as storage buffers, like this:

**index.html (Compute createShaderModule call)**

```
@group(0) @binding(0) var<uniform> grid: vec2f;
    
// New lines
@group(0) @binding(1) var<storage> cellStateIn: array<u32>;
@group(0) @binding(2) var<storage, read_write> cellStateOut: array<u32>;

@compute @workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE})
fn computeMain(@builtin(global_invocation_id) cell: vec3u) {

}
```

Note that the first storage buffer is declared with `var<storage>`, which makes it read-only, but the second storage buffer is declared with `var<storage, read_write>`. This allows you to both read and write to the buffer, using that buffer as the output for your compute shader. (There is no write-only storage mode in WebGPU).

Next, you need to have a way to map your cell index into the linear storage array. This is basically the opposite of what you did in the vertex shader, where you took the linear `instance_index` and mapped it to a 2D grid cell. (As a reminder, your algorithm for that was `vec2f(i % grid.x, floor(i / grid.x))`.)

7. Write a function to go in the other direction. It takes the cell's Y value, multiplies it by the grid width, and then adds the cell's X value.

**index.html (Compute createShaderModule call)**

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@group(0) @binding(1) var<storage> cellStateIn: array<u32>;
@group(0) @binding(2) var<storage, read_write> cellStateOut: array<u32>;

// New function   
fn cellIndex(cell: vec2u) -> u32 {
  return cell.y * u32(grid.x) + cell.x;
}

@compute @workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE})
fn computeMain(@builtin(global_invocation_id) cell: vec3u) {
  
}
```

And, finally, to see that it's working, implement a really simple algorithm: if a cell is currently on, it turns off, and vice versa. It's not the Game of Life yet, but it's enough to show that the compute shader is working.

8. Add the simple algorithm, like this:

**index.html (Compute createShaderModule call)**

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@group(0) @binding(1) var<storage> cellStateIn: array<u32>;
@group(0) @binding(2) var<storage, read_write> cellStateOut: array<u32>;
   
fn cellIndex(cell: vec2u) -> u32 {
  return cell.y * u32(grid.x) + cell.x;
}

@compute @workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE})
fn computeMain(@builtin(global_invocation_id) cell: vec3u) {
  // New lines. Flip the cell state every step.
  if (cellStateIn[cellIndex(cell.xy)] == 1) {
    cellStateOut[cellIndex(cell.xy)] = 0;
  } else {
    cellStateOut[cellIndex(cell.xy)] = 1;
  }
}
```



**Note:** The `cell.xy` syntax here is a shorthand known as *swizzling*. It's equivalent to saying `vec2(cell.x, cell.y)` and is an easy way to get the components you need out of a vector in a different configuration. Swizzling is very flexible! You can put the components of the vector (up to four of them) in any order and repeat them as needed to do things like `cell.zyx` or `cell.yyyy`.

And that's it for your compute shader—for now! But before you can see the results, there are a few more changes that you need to make.

<a id="use-bind-group-and-pipeline-layouts"></a>

## **Use Bind Group and Pipeline Layouts**

One thing that you might notice from the above shader is that it largely uses the same inputs (uniforms and storage buffers) as your render pipeline. So you might think that you can simply use the same bind groups and be done with it, right? The good news is that you can! It just takes a bit more manual setup to be able to do that.

Any time that you create a bind group, you need to provide a [`GPUBindGroupLayout`](https://gpuweb.github.io/gpuweb/#gpubindgrouplayout). Previously, you got that layout by calling `getBindGroupLayout()` on the render pipeline, which in turn created it automatically because you supplied `layout: "auto"` when you created it. That approach works well when you only use a single pipeline, but if you have multiple pipelines that want to share resources, you need to create the layout explicitly, and then provide it to both the bind group and pipelines.

To help understand why, consider this: in your render pipelines you use a single uniform buffer and a single storage buffer, but in the compute shader you just wrote, you need a second storage buffer. Because the two shaders use the same `@binding` values for the uniform and first storage buffer, you can share those between pipelines, and the render pipeline ignores the second storage buffer, which it doesn't use. You want to create a layout that describes *all* of the resources that are present in the bind group, not just the ones used by a specific pipeline.

1. To create that layout, call [`device.createBindGroupLayout()`](https://gpuweb.github.io/gpuweb/#dom-gpudevice-createbindgrouplayout):

**index.html**

```
// Create the bind group layout and pipeline layout.
const bindGroupLayout = device.createBindGroupLayout({
  label: "Cell Bind Group Layout",
  entries: [{
    binding: 0,
    // Add GPUShaderStage.FRAGMENT here if you are using the `grid` uniform in the fragment shader.
    visibility: GPUShaderStage.VERTEX | GPUShaderStage.COMPUTE,
    buffer: {} // Grid uniform buffer
  }, {
    binding: 1,
    visibility: GPUShaderStage.VERTEX | GPUShaderStage.COMPUTE,
    buffer: { type: "read-only-storage"} // Cell state input buffer
  }, {
    binding: 2,
    visibility: GPUShaderStage.COMPUTE,
    buffer: { type: "storage"} // Cell state output buffer
  }]
});
```

This is similar in structure to creating the bind group itself, in that you describe a list of *`entries`*. The difference is that you describe what type of resource the entry must be and how it's used rather than providing the resource itself.

In each entry, you give the *`binding`* number for the resource, which (as you learned when you created the bind group) matches the `@binding` value in the shaders. You also provide the *`visibility`*, which are [`GPUShaderStage`](https://gpuweb.github.io/gpuweb/#typedefdef-gpushaderstageflags) flags that indicate which shader stages can use the resource. You want both the uniform and first storage buffer to be accessible in the vertex and compute shaders, but the second storage buffer only needs to be accessible in compute shaders.

**Note:** You may need to make resources accessible to fragment shaders by adding *`GPUShaderStage.FRAGMENT`* for the uniform and first storage buffer if your fragment shader code uses the uniform *`grid`* variable as explained in the previous section.

Finally, you indicate what type of resource is being used. This is a different dictionary key, depending on what you need to expose. Here, all three resources are buffers, so you use the *`buffer`* key to define the options for each. Other options include things like *`texture`* or *`sampler`*, but you don't need those here.

In the buffer dictionary, you set options like what `type` of buffer is used. The default is `"uniform"`, so you can leave the dictionary empty for binding 0. (You do have to at least set `buffer: {}`, though, so that the entry is identified as a buffer.) Binding 1 is given a type of `"read-only-storage"` because you don't use it with `read_write` access in the shader, and binding 2 has a type of `"storage"` because you *do* use it with `read_write` access!

Once the `bindGroupLayout` is created, you can pass it in when creating your bind groups rather than querying the bind group from the pipeline. Doing so means that you need to add a new storage buffer entry to each bind group in order to match the layout you just defined.

2. Update the bind group creation, like this:

**index.html**

```
// Create a bind group to pass the grid uniforms into the pipeline
const bindGroups = [
  device.createBindGroup({
    label: "Cell renderer bind group A",
    layout: bindGroupLayout, // Updated Line
    entries: [{
      binding: 0,
      resource: { buffer: uniformBuffer }
    }, {
      binding: 1,
      resource: { buffer: cellStateStorage[0] }
    }, {
      binding: 2, // New Entry
      resource: { buffer: cellStateStorage[1] }
    }],
  }),
  device.createBindGroup({
    label: "Cell renderer bind group B",
    layout: bindGroupLayout, // Updated Line

    entries: [{
      binding: 0,
      resource: { buffer: uniformBuffer }
    }, {
      binding: 1,
      resource: { buffer: cellStateStorage[1] }
    }, {
      binding: 2, // New Entry
      resource: { buffer: cellStateStorage[0] }
    }],
  }),
];
```

And now that the bind group has been updated to use this explicit bind group layout, you need to update the render pipeline to use the same thing.

3. Create a [`GPUPipelineLayout`](https://gpuweb.github.io/gpuweb/#gpupipelinelayout).

<a id="index.html_27"></a>

### **index.html**

```
const pipelineLayout = device.createPipelineLayout({
  label: "Cell Pipeline Layout",
  bindGroupLayouts: [ bindGroupLayout ],
});
```

A pipeline layout is a list of bind group layouts (in this case, you have one) that one or more pipelines use. The order of the bind group layouts in the array needs to correspond with the `@group` attributes in the shaders. (This means that `bindGroupLayout` is associated with `@group(0)`.)

4. Once you have the pipeline layout, update the render pipeline to use it instead of `"auto"`.

<a id="index.html_28"></a>

### **index.html**

```
const cellPipeline = device.createRenderPipeline({
  label: "Cell pipeline",
  layout: pipelineLayout, // Updated!
  vertex: {
    module: cellShaderModule,
    entryPoint: "vertexMain",
    buffers: [vertexBufferLayout]
  },
  fragment: {
    module: cellShaderModule,
    entryPoint: "fragmentMain",
    targets: [{
      format: canvasFormat
    }]
  }
});
```



<a id="create-the-compute-pipeline"></a>

## **Create the compute pipeline**

Just like you need a render pipeline to use your vertex and fragment shaders, you need a compute pipeline to use your compute shader. Fortunately, compute pipelines are *far* less complicated than render pipelines, as they don't have any state to set, only the shader and layout.

- Create a compute pipeline with the following code:

**index.html**

```
// Create a compute pipeline that updates the game state.
const simulationPipeline = device.createComputePipeline({
  label: "Simulation pipeline",
  layout: pipelineLayout,
  compute: {
    module: simulationShaderModule,
    entryPoint: "computeMain",
  }
});
```

Notice that you pass in the new `pipelineLayout` instead of `"auto"`, just like in the updated render pipeline, which ensures that both your render pipeline and your compute pipeline can use the same bind groups.

<a id="compute-passes"></a>

## **Compute passes**

This brings you to the point of actually making use of the compute pipeline! Given that you do your rendering in a render pass, you can probably guess that you need to do compute work in a compute pass. Compute and render work can both happen in the same command encoder, so you want to shuffle your `updateGrid` function a bit.

1. Move the encoder creation to the top of the function, and then begin a compute pass with it (before the `step++`).

**index.html**

```
// In updateGrid()
// Move the encoder creation to the top of the function.
const encoder = device.createCommandEncoder();

const computePass = encoder.beginComputePass();

// Compute work will go here...

computePass.end();

// Existing lines
step++; // Increment the step count
  
// Start a render pass...
```

Just like compute pipelines, compute passes are much simpler to kick off than their rendering counterparts because you don't need to worry about any attachments.

You want to do the compute pass before the render pass because it allows the render pass to immediately use the latest results from the compute pass. That's also the reason that you increment the `step` count between the passes, so that the output buffer of the compute pipeline becomes the input buffer for the render pipeline.

2. Next, set the pipeline and bind group inside the compute pass, using the same pattern for switching between bind groups as you do for the rendering pass.

**index.html**

```
const computePass = encoder.beginComputePass();

// New lines
computePass.setPipeline(simulationPipeline);
computePass.setBindGroup(0, bindGroups[step % 2]);

computePass.end();
```

3. Finally, instead of drawing like in a render pass, you dispatch the work to the compute shader, telling it how many workgroups you want to execute on each axis.

**index.html**

```
const computePass = encoder.beginComputePass();

computePass.setPipeline(simulationPipeline);
computePass.setBindGroup(0, bindGroups[step % 2]);

// New lines
const workgroupCount = Math.ceil(GRID_SIZE / WORKGROUP_SIZE);
computePass.dispatchWorkgroups(workgroupCount, workgroupCount);

computePass.end();
```

Something *very important* to note here is that the number you pass into `dispatchWorkgroups()` is **not** the number of invocations! Instead, it's the number of workgroups to execute, as defined by the `@workgroup_size` in your shader.

If you want the shader to execute 32x32 times in order to cover your entire grid, and your workgroup size is 8x8, you need to dispatch 4x4 workgroups (4 \* 8 = 32). That's why you divide the grid size by the workgroup size and pass that value into `dispatchWorkgroups()`.

**Note:** Compute work can only be dispatched by workgroup, so if you have a workload that's not an even divisor of the workgroup size, you have two options. You can either change the workgroup size (which has to be done at shader module creation time, and has relatively high overhead), or you can round up the number of workgroups you dispatch, and then, in the shader, check to see if you're over the desired `global_invocation_id` and need to return early.

Now you can refresh the page again, and you should see that the grid inverts itself with each update.

![Diagonal stripes of colorful squares going from bottom left to top right against a dark blue background. ](<your-first-webgpu-app/1f5218012d24c59a53bd1f0c.png>) ![Diagonal stripes of colorful squares two squares wide going from bottom left to top right against a dark blue background. The inversion of the previous image.](<your-first-webgpu-app/f0747aef2decaddc35409980.png>)

<a id="implement-the-algorithm-for-the-game-of-life"></a>

## **Implement the algorithm for the Game of Life**

Before you update the compute shader to implement the final algorithm, you want to go back to the code that's initializing the storage buffer content and update it to produce a random buffer on each page load. (Regular patterns don't make for very interesting Game of Life starting points.) You can randomize the values however you want, but there's an easy way to start that gives reasonable results.

1. To start each cell in a random state, update the `cellStateArray` initialization to the following code:

**index.html**

```
// Set each cell to a random state, then copy the JavaScript array 
// into the storage buffer.
for (let i = 0; i < cellStateArray.length; ++i) {
  cellStateArray[i] = Math.random() > 0.6 ? 1 : 0;
}
device.queue.writeBuffer(cellStateStorage[0], 0, cellStateArray);
```

Now you can finally implement the logic for the Game of Life simulation. After everything it took to get here, the shader code may be disappointingly simple!

First, you need to know for any given cell how many of its neighbors are active. You don't care about which ones are active, only the count.

2. To make getting neighboring cell data easier, add a `cellActive` function that returns the `cellStateIn` value of the given coordinate.

**index.html (Compute createShaderModule call)**

```
fn cellActive(x: u32, y: u32) -> u32 {
  return cellStateIn[cellIndex(vec2(x, y))];
}
```

The `cellActive` function returns one if the cell is active, so adding the return value of calling `cellActive` for all eight surrounding cells gives you how many neighboring cells are active.

3. Find the number of active neighbors, like this:

**index.html (Compute createShaderModule call)**

```
fn computeMain(@builtin(global_invocation_id) cell: vec3u) {
  // New lines:
  // Determine how many active neighbors this cell has.
  let activeNeighbors = cellActive(cell.x+1, cell.y+1) +
                        cellActive(cell.x+1, cell.y) +
                        cellActive(cell.x+1, cell.y-1) +
                        cellActive(cell.x, cell.y-1) +
                        cellActive(cell.x-1, cell.y-1) +
                        cellActive(cell.x-1, cell.y) +
                        cellActive(cell.x-1, cell.y+1) +
                        cellActive(cell.x, cell.y+1);
```

This leads to a minor problem, though: what happens when the cell you're checking is off the edge of the board? According to your `cellIndex()` logic right now, it either overflows to the next or previous row, or runs off the edge of the buffer!

**Caution:** Unlike some languages, indexing [outside the bounds](https://gpuweb.github.io/gpuweb/wgsl/#out-of-bounds-access-sec) of a buffer in WGSL isn't *unsafe*, but it is unpredictable. The language makes sure you still get *some* value from the buffer, so you can't accidentally end up reading data from another process or anything like that, but it's almost certain to not give you the results you want. And the behavior may be different, depending on the platform and browser you use, so you should never depend on out-of-bounds accesses.

For the Game of Life, a common and easy way to resolve this is to have cells on the edge of the grid treat cells on the opposite edge of the grid as their neighbors, creating a kind of wrap-around effect.

4. Support grid wrap-around with a minor change to the `cellIndex()` function.

**index.html (Compute createShaderModule call)**

```
fn cellIndex(cell: vec2u) -> u32 {
  return (cell.y % u32(grid.y)) * u32(grid.x) +
         (cell.x % u32(grid.x));
}
```

By using the `%` operator to wrap the cell X and Y when it extends past the grid size, you ensure that you never access outside the storage buffer bounds. With that, you can rest assured that the `activeNeighbors` count is predictable.

Then you apply one of four rules:

- Any cell with fewer than two neighbors becomes inactive.
- Any active cell with two or three neighbors stays active.
- Any inactive cell with exactly three neighbors becomes active.
- Any cell with more than three neighbors becomes inactive.

You can do this with a series of if statements, but WGSL also supports switch statements, which are a good fit for this logic.

5. Implement the Game of Life logic, like this:

**index.html (Compute createShaderModule call)**

```
let i = cellIndex(cell.xy);

// Conway's game of life rules:
switch activeNeighbors {
  case 2: { // Active cells with 2 neighbors stay active.
    cellStateOut[i] = cellStateIn[i];
  }
  case 3: { // Cells with 3 neighbors become or stay active.
    cellStateOut[i] = 1;
  }
  default: { // Cells with < 2 or > 3 neighbors become inactive.
    cellStateOut[i] = 0;
  }
}
```

For reference, the final compute shader module call now looks like this:

**index.html**

```
const simulationShaderModule = device.createShaderModule({
  label: "Life simulation shader",
  code: `
    @group(0) @binding(0) var<uniform> grid: vec2f;

    @group(0) @binding(1) var<storage> cellStateIn: array<u32>;
    @group(0) @binding(2) var<storage, read_write> cellStateOut: array<u32>;

    fn cellIndex(cell: vec2u) -> u32 {
      return (cell.y % u32(grid.y)) * u32(grid.x) +
              (cell.x % u32(grid.x));
    }

    fn cellActive(x: u32, y: u32) -> u32 {
      return cellStateIn[cellIndex(vec2(x, y))];
    }

    @compute @workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE})
    fn computeMain(@builtin(global_invocation_id) cell: vec3u) {
      // Determine how many active neighbors this cell has.
      let activeNeighbors = cellActive(cell.x+1, cell.y+1) +
                            cellActive(cell.x+1, cell.y) +
                            cellActive(cell.x+1, cell.y-1) +
                            cellActive(cell.x, cell.y-1) +
                            cellActive(cell.x-1, cell.y-1) +
                            cellActive(cell.x-1, cell.y) +
                            cellActive(cell.x-1, cell.y+1) +
                            cellActive(cell.x, cell.y+1);

      let i = cellIndex(cell.xy);

      // Conway's game of life rules:
      switch activeNeighbors {
        case 2: {
          cellStateOut[i] = cellStateIn[i];
        }
        case 3: {
          cellStateOut[i] = 1;
        }
        default: {
          cellStateOut[i] = 0;
        }
      }
    }
  `
});
```

And... that's it! You're done! Refresh your page and watch your newly built cellular automaton grow!

![Screenshot of an example state from the Game of Life simulation, with colorful cells rendered against a dark blue background.](<your-first-webgpu-app/25daed54056cbbf89b927104.png>)

<a id="9-congratulations"></a>

Source: [https://codelabs.developers.google.com/your-first-webgpu-app#7](<https://codelabs.developers.google.com/your-first-webgpu-app#7>)
