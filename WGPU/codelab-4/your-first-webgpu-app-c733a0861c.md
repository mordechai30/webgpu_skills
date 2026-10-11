# Your first WebGPU app

## [5. Draw a grid](https://codelabs.developers.google.com/your-first-webgpu-app#4)

First, take a moment to congratulate yourself! Getting the first bits of geometry on screen is often one of the hardest steps with most GPU APIs. Everything you do from here can be done in smaller steps, making it easier to verify your progress as you go.

In this section, you learn:

- How to pass variables (called uniforms) to the shader from JavaScript.
- How to use uniforms to change the rendering behavior.
- How to use instancing to draw many different variants of the same geometry.

<a id="define-the-grid"></a>

## **Define the grid**

In order to render a grid, you need to know a very fundamental piece of information about it. How many cells does it contain, both in width and height? This is up to you as the developer, but to keep things a bit easier, treat the grid as a square (same width and height) and use a size that's a power of two. (That makes some of the math easier later.) You want to make it bigger eventually, but for the rest of this section, set your grid size to 4x4 because it makes it easier to demonstrate some of the math used in this section. Scale it up after!

- Define the grid size by adding a constant to the top of your JavaScript code.

<a id="index.html_21"></a>

### **index.html**

```
const GRID_SIZE = 4;
```

Next, you need to update how you render your square so that you can fit `GRID_SIZE` times `GRID_SIZE` of them on the canvas. That means the square needs to be a lot smaller, and there needs to be a lot of them.

Now, one way you *could* approach this is by making your vertex buffer significantly bigger and defining `GRID_SIZE` times `GRID_SIZE` worth of squares inside it at the right size and position. The code for that wouldn't be too bad, in fact! Just a couple of for loops and a bit of math. But that's also not making the best use of the GPU and using more memory than necessary to achieve the effect. This section looks at a more GPU-friendly approach.

<a id="create-a-uniform-buffer"></a>

## **Create a uniform buffer**

First, you need to communicate the grid size you've chosen to the shader, since it uses that to change how things display. You could just hard-code the size into the shader, but then that means that any time you want to change the grid size you have to re-create the shader and render pipeline, which is expensive. A better way is to provide the grid size to the shader as *uniforms*.

You learned earlier that a different value from the vertex buffer is passed to every invocation of a vertex shader. A uniform is a value from a buffer that is the same for every invocation. They're useful for communicating values that are common for a piece of geometry (like its position), a full frame of animation (like the current time), or even the entire lifespan of the app (like a user preference).

- Create a uniform buffer by adding the following code:

<a id="index.html_22"></a>

### **index.html**

```
// Create a uniform buffer that describes the grid.
const uniformArray = new Float32Array([GRID_SIZE, GRID_SIZE]);
const uniformBuffer = device.createBuffer({
  label: "Grid Uniforms",
  size: uniformArray.byteLength,
  usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});
device.queue.writeBuffer(uniformBuffer, 0, uniformArray);
```

This should look very familiar, because it's almost exactly the same code that you used to create the vertex buffer earlier! That's because uniforms are communicated to the WebGPU API through the same GPUBuffer objects that vertices are, with the main difference being that the *`usage`* this time includes `GPUBufferUsage.UNIFORM` instead of `GPUBufferUsage.VERTEX`.

**Note:** You could use a [`Uint32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint32Array) here instead, as the grid size values aren't fractional, but in this specific case it would make the upcoming shader code changes a bit messier because you'd end up casting the values to floats in several places anyway. Just keep in mind that a float isn't the only type of data you can place in a `GPUBuffer`!



<a id="access-uniforms-in-a-shader"></a>

## **Access uniforms in a shader**

- Define a uniform by adding the following code:

<a id="index.html-createshadermodule-call"></a>

### **index.html (createShaderModule call)**

```
// At the top of the `code` string in the createShaderModule() call
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(@location(0) pos: vec2f) ->
  @builtin(position) vec4f {
  return vec4f(pos / grid, 0, 1);
}

// ...fragmentMain is unchanged 
```

This defines a uniform in your shader called `grid`, which is a 2D float vector that matches the array that you just copied into the uniform buffer. It also specifies that the uniform is bound at `@group(0)` and `@binding(0)`. You'll learn what those values mean in a moment.

Then, elsewhere in the shader code, you can use the grid vector however you need. In this code you divide the vertex position by the grid vector. Since `pos` is a 2D vector and `grid` is a 2D vector, WGSL performs a component-wise division. In other words, the result is the same as saying `vec2f(pos.x / grid.x, pos.y / grid.y)`.

These types of vector operations are very common in GPU shaders since many rendering and compute techniques rely on them!

What this means in your case is that (if you used a grid size of 4) the square that you render would be one-fourth of its original size. That's perfect if you want to fit four of them to a row or column!

<a id="create-a-bind-group"></a>

## **Create a Bind Group**

Declaring the uniform in the shader doesn't connect it with the buffer that you created, though. In order to do that, you need to create and set a *bind group*.

A bind group is a collection of resources that you want to make accessible to your shader at the same time. It can include several types of buffers, like your uniform buffer, and other resources like textures and samplers that are not covered here but are common parts of WebGPU rendering techniques.

- Create a bind group with your uniform buffer by adding the following code after the creation of the uniform buffer and render pipeline:

<a id="index.html_23"></a>

### **index.html**

```
const bindGroup = device.createBindGroup({
  label: "Cell renderer bind group",
  layout: cellPipeline.getBindGroupLayout(0),
  entries: [{
    binding: 0,
    resource: { buffer: uniformBuffer }
  }],
});
```

In addition to your now-standard *`label`*, you also need a *`layout`* that describes which types of resources this bind group contains. This is something that you dig into further in a future step, but for the moment you can happily ask your pipeline for the bind group layout because you created the pipeline with `layout: "auto"`. That causes the pipeline to create bind group layouts automatically from the bindings that you declared in the shader code itself. In this case, you ask it to `getBindGroupLayout(0)`, where the `0` corresponds to the `@group(0)` that you typed in the shader.

After specifying the layout, you provide an array of *`entries`*. Each entry is a dictionary with at least the following values:

- **`binding`**, which corresponds with the `@binding()` value you entered in the shader. In this case, `0`.
- **`resource`**, which is the actual resource that you want to expose to the variable at the specified binding index. In this case, your uniform buffer.

The function returns a [`GPUBindGroup`](https://gpuweb.github.io/gpuweb/#gpubindgroup), which is an opaque, immutable handle. You can't change the resources that a bind group points to after it's been created, though you *can* change the contents of those resources. For example, if you change the uniform buffer to contain a new grid size, that is reflected by future draw calls using this bind group.

<a id="bind-the-bind-group"></a>

## **Bind the bind group**

Now that the bind group is created, you still need to tell WebGPU to use it when drawing. Fortunately this is pretty simple.

1. Hop back down to the render pass and add this new line before the `draw()` method:

<a id="index.html_24"></a>

### **index.html**

```
pass.setPipeline(cellPipeline);
pass.setVertexBuffer(0, vertexBuffer);

pass.setBindGroup(0, bindGroup); // New line!

pass.draw(vertices.length / 2);
```

The `0` passed as the first argument corresponds to the `@group(0)` in the shader code. You're saying that each `@binding` that's part of `@group(0)` uses the resources in this bind group.

And now the uniform buffer is exposed to your shader!

2. Refresh your page, and then you should see something like this:

![A small red square in the center of a dark blue background.](<your-first-webgpu-app/77560894e806f8afc2a6d3c1.png>)

Hooray! Your square is now one-fourth the size it was before! That's not much, but it shows that your uniform is actually applied and that the shader can now access the size of your grid.

<a id="manipulate-geometry-in-the-shader"></a>

## **Manipulate geometry in the shader**

So now that you can reference the grid size in the shader, you can start doing some work to manipulate the geometry you're rendering to fit your desired grid pattern. To do that, consider exactly what you want to achieve.

You need to conceptually divide up your canvas into individual cells. In order to keep the convention that the X axis increases as you move right and the Y axis increases as you move up, say that the first cell is in the bottom left corner of the canvas. That gives you a layout that looks like this, with your current square geometry in the middle:

![An illustration of the conceptual grid the Normalized Device Coordinate space will be divided when visualizing each cell with the currently rendered square geometry at it's center.](<your-first-webgpu-app/bd2c048827357fdaf6896e8f.png>)

Your challenge is to find a method in the shader that lets you position the square geometry in any of those cells given the cell coordinates.

First, you can see that your square isn't nicely aligned with any of the cells because it was defined to surround the center of the canvas. You'd want to have the square shifted by half a cell so that it would line up nicely inside them.

One way you could fix this is to update the square's vertex buffer. By shifting the vertices so that the bottom-left corner is at, for example, (0.1, 0.1) instead of (-0.8, -0.8), you'd move this square to line up with the cell boundaries more nicely. But, since you have full control over how the vertices are processed in your shader, it's just as easy to simply nudge them into place using the shader code!

1. Alter the vertex shader module with the following code:

<a id="index.html-createshadermodule-call_1"></a>

### index.html (createShaderModule call)

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(@location(0) pos: vec2f) ->
  @builtin(position) vec4f {

  // Add 1 to the position before dividing by the grid size.
  let gridPos = (pos + 1) / grid;

  return vec4f(gridPos, 0, 1);
}
```



**Caution:** For the sake of making it easier to read, the position calculation is broken out into its own variable. Notice that it's declared using `let`, but be aware that `let` in WGSL has a different meaning than `let` in JavaScript! In WGSL, `let` behaves more like JavaScript's `const` in that it indicates that the value of the variable won't change after assignment. If you do need to change a variable after declaring it in WGSL, use `var` instead.



**Note:** Since `pos` is a `vec2f` here, adding `1` does a component-wise addition. It's the same as saying `pos + vec2f(1, 1)`. The same works for subtraction, multiplication, and division!

This moves every vertex up and to the right by one (which, remember, is half of the clip space) *before* dividing it by the grid size. The result is a nicely grid-aligned square just off of the origin.

![A visualization of the canvas conceptually divided into a 4x4 grid with a red square in cell (2, 2)](<your-first-webgpu-app/cc36634013e66e204dab0ae8.png>)

Next, because your canvas' coordinate system places (0, 0) in the center and (-1, -1) in the lower left, and you want (0, 0) to be in the lower left, you need to translate your geometry's position by (-1, -1) *after* dividing by the grid size in order to move it into that corner.

2. Translate your geometry's position, like this:

<a id="index.html-createshadermodule-call_2"></a>

### index.html (createShaderModule call)

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(@location(0) pos: vec2f) ->
  @builtin(position) vec4f {

  // Subtract 1 after dividing by the grid size.
  let gridPos = (pos + 1) / grid - 1;

  return vec4f(gridPos, 0, 1); 
}
```

And now your square is nicely positioned in cell (0, 0)!

![A visualization of the canvas conceptually divided into a 4x4 grid with a red square in cell (0, 0)](<your-first-webgpu-app/26f1cfb83e91527d80bb653c.png>)

What if you want to place it in a different cell? Figure that out by declaring a `cell` vector in your shader and populating it with a static value like `let cell = vec2f(1, 1)`.

If you add that to the `gridPos`, it undoes the `- 1` in the algorithm, so that's not what you want. Instead, you want to move the square only by one grid unit (one-fourth of the canvas) for each cell. Sounds like you need to do another divide by `grid`!

3. Change your grid positioning, like this:

<a id="index.html-createshadermodule-call_3"></a>

### index.html (createShaderModule call)

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(@location(0) pos: vec2f) ->
  @builtin(position) vec4f {

  let cell = vec2f(1, 1); // Cell(1,1) in the image above
  let cellOffset = cell / grid; // Compute the offset to cell
  let gridPos = (pos + 1) / grid - 1 + cellOffset; // Add it here!

  return vec4f(gridPos, 0, 1);
}
```

If you refresh now, you see the following:

![A visualization of the canvas conceptually divided into a 4x4 grid with a red square centered between cell (0, 0), cell (0, 1), cell (1, 0), and cell (1, 1)](<your-first-webgpu-app/5857733d15102c26c2ab27f2.png>)

Hm. Not quite what you wanted.

The reason for this is that since the canvas coordinates go from -1 to +1, it's *actually 2 units across*. That means if you want to move a vertex one-fourth of the canvas over, you have to move it 0.5 units. This is an easy mistake to make when reasoning with GPU coordinates! Fortunately, the fix is just as easy.

4. Multiply your offset by 2, like this:

<a id="index.html-createshadermodule-call_4"></a>

### index.html (createShaderModule call)

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(@location(0) pos: vec2f) ->
  @builtin(position) vec4f {

  let cell = vec2f(1, 1);
  let cellOffset = cell / grid * 2; // Updated
  let gridPos = (pos + 1) / grid - 1 + cellOffset;

  return vec4f(gridPos, 0, 1);
}
```

And this gives you exactly what you want.

![A visualization of the canvas conceptually divided into a 4x4 grid with a red square in cell (1, 1)](<your-first-webgpu-app/f59fc3943f5456cfef57bdf5.png>)

The screenshot looks like this:

![Screenshot of a red square on a dark blue background. The red square in drawn in the same position as described in the previous diagram, but without the grid overlay.](<your-first-webgpu-app/c1cd562083f75b3907966ff7.png>)

Furthermore, you can now set `cell` to any value within the grid bounds, and then refresh to see the square render in the desired location.

<a id="draw-instances"></a>

## **Draw instances**

Now that you can place the square where you want it with a bit of math, the next step is to render one square in each cell of the grid.

One way you could approach it is to write cell coordinates to a uniform buffer, then call *draw* once for each square in the grid, updating the uniform every time. That would be very slow, however, since the GPU has to wait for the new coordinate to be written by JavaScript every time. One of the keys to getting good performance out of the GPU is to minimize the time it spends waiting on other parts of the system!

Instead, you can use a technique called instancing. Instancing is a way to tell the GPU to draw multiple copies of the same geometry with a single call to `draw`, which is much faster than calling `draw` once for every copy. Each copy of the geometry is referred to as an *instance*.

1. To tell the GPU that you want enough instances of your square to fill the grid, add one argument to your existing draw call:

<a id="index.html_25"></a>

### **index.html**

```
pass.draw(vertices.length / 2, GRID_SIZE * GRID_SIZE);
```

This tells the system that you want it to draw the six (`vertices.length / 2`) vertices of your square 16 (`GRID_SIZE * GRID_SIZE`) times. But if you refresh the page, you still see the following:

![An identical image to the previous diagram, to indicate that nothing changed.](<your-first-webgpu-app/c1cd562083f75b3907966ff7.png>)

Why? Well, it's because you draw all 16 of those squares in the same spot. Your need to have some additional logic in the shader that repositions the geometry on a per-instance basis.

In the shader, in addition to the vertex attributes like `pos` that come from your vertex buffer, you can also access what are known as WGSL's [*built-in values*](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values). These are values that are calculated by WebGPU, and one such value is the `instance_index`. The `instance_index` is an unsigned 32-bit number from `0` to `number of instances - 1` that you can use as part of your shader logic. Its value is the same for every vertex processed that's part of the same instance. That means your vertex shader gets called six times with an `instance_index` of `0`, once for each position in your vertex buffer. Then six more times with an `instance_index` of `1`, then six more with `instance_index` of `2`, and so on.

To see this in action, you have to add the `instance_index` built-in to your shader inputs. Do this in the same way as the position, but instead of tagging it with a `@location` attribute, use `@builtin(instance_index)`, and then name the argument whatever you want. (You can call it `instance` to match the example code.) Then use it as part of the shader logic!

2. Use `instance` in place of the cell coordinates:

**index.html**

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(@location(0) pos: vec2f,
              @builtin(instance_index) instance: u32) ->
  @builtin(position) vec4f {
  
  let i = f32(instance); // Save the instance_index as a float
  let cell = vec2f(i, i); // Updated
  let cellOffset = cell / grid * 2;
  let gridPos = (pos + 1) / grid - 1 + cellOffset;

  return vec4f(gridPos, 0, 1);
}
```



**Caution:** This is an area where WGSL's strong typing may trip up developers who are more familiar with JavaScript than other strongly typed languages. The `instance_index` is a `u32`, or unsigned 32-bit integer. But you want `cell` to be a vector of floating point values because that's how the position math is done. As a result, you have to explicitly tell the shader to treat `instance` as a float when creating the `cell` vector by wrapping it in a `f32()` function. This is known as *casting* the type.

If you refresh now you see that you do indeed have more than one square! But you can't see all 16 of them.

![Four red squares in a diagonal line from the lower left corner to the top right corner against a dark blue background.](<your-first-webgpu-app/d56ef7f4fd7dc40c37f5964e.png>)

That's because the cell coordinates you generate are (0, 0), (1, 1), (2, 2)... all the way to (15, 15), but only the first four of those fit on the canvas. To make the grid that you want, you need to transform the `instance_index` such that each index maps to a unique cell within your grid, like this:

![A visualization of the canvas conceptually divided into a 4x4 grid with each cell also corresponding to a linear instance index.](<your-first-webgpu-app/9709f94874866d543146c048.png>)

The math for that is reasonably straightforward. For each cell's X value, you want the [modulo](https://en.wikipedia.org/wiki/Modulo) of the `instance_index` and the grid width, which you can perform in WGSL with the `%` operator. And for each cell's Y value you want the `instance_index` divided by the grid width, discarding any fractional remainder. You can do that with WGSL's [`floor()`](https://gpuweb.github.io/gpuweb/wgsl/#floor-builtin) function.

3. Change the calculations, like this:

**index.html**

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(@location(0) pos: vec2f,
              @builtin(instance_index) instance: u32) ->
  @builtin(position) vec4f {

  let i = f32(instance);
  // Compute the cell coordinate from the instance_index
  let cell = vec2f(i % grid.x, floor(i / grid.x));

  let cellOffset = cell / grid * 2;
  let gridPos = (pos + 1) / grid - 1 + cellOffset;

  return vec4f(gridPos, 0, 1);
}
```

After making that update to the code you have the long-awaited grid of squares at last!

![Four rows of four columns of red squares on a dark blue background.](<your-first-webgpu-app/a994400f44405f13e6a6787c.png>)

4. And now that it's working, go back and crank up the grid size!

<a id="index.html_26"></a>

### **index.html**

```
const GRID_SIZE = 32;
```

![32 rows of 32 columns of red squares on a dark blue background.](<your-first-webgpu-app/a308c78f95bc8d7372c9785f.png>)

Tada! You can actually make this grid really, *really* big now and your average GPU handles it just fine. You'll stop seeing the individual squares long before you run into any GPU performance bottlenecks.

**Note:** You might be thinking, "That could have all been done with one line of Canvas 2D code in a nested loop!" And, well... yeah! It could have! And if that level of rendering is all you're interested in then you're probably better off sticking with a simpler API like Canvas 2D. This codelab is focused on introducing core WebGPU concepts, and pushing the limits of the API's rendering capabilities is way, *way* out of scope.

It's worth noting that if you have a use case where you want to use Canvas 2D for more simplistic rendering needs like this but still want to take advantage of WebGPU's compute capabilities, you can use the APIs cooperatively! It requires more data copies than if everything is done in WebGPU. See [WebGPU — All of the cores, none of the canvas](https://surma.dev/things/webgpu/) as an example of using WebGPU and Canvas 2D together.

<a id="6-extra-credit-make-it-more-colorful"></a>

Source: [https://codelabs.developers.google.com/your-first-webgpu-app#4](<https://codelabs.developers.google.com/your-first-webgpu-app#4>)
