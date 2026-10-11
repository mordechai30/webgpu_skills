# Your first WebGPU app

## [6. Extra credit: make it more colorful!](https://codelabs.developers.google.com/your-first-webgpu-app#5)

At this point, you can easily skip to the next section since you've laid the groundwork for the rest of the codelab. But while the grid of squares all sharing the same color is serviceable, it's not exactly exciting, is it? Fortunately you can make things a bit brighter with a little more math and shader code!

<a id="use-structs-in-shaders"></a>

## Use structs in shaders

Until now, you've passed one piece of data out of the vertex shader: the transformed position. But you can actually return a lot more data from the vertex shader and then use it in the fragment shader!

The only way to pass data out of the vertex shader is by returning it. A vertex shader is always required to return a position, so if you want to return any other data along with it, you need to place it in a struct. Structs in WGSL are named object types that contain one or more named properties. The properties can be marked up with attributes like `@builtin` and `@location` too. You declare them outside of any functions, and then you can pass instances of them in and out of functions, as needed. For example, consider your current vertex shader:

<a id="index.html-createshadermodule-call_5"></a>

### index.html (createShaderModule call)

```
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(@location(0) pos: vec2f,
              @builtin(instance_index) instance: u32) -> 
  @builtin(position) vec4f {

  let i = f32(instance);
  let cell = vec2f(i % grid.x, floor(i / grid.x));
  let cellOffset = cell / grid * 2;
  let gridPos = (pos + 1) / grid - 1 + cellOffset;
  
  return  vec4f(gridPos, 0, 1);
}
```

- Express the same thing using structs for the function input and output:

<a id="index.html-createshadermodule-call_6"></a>

### index.html (createShaderModule call)

```
struct VertexInput {
  @location(0) pos: vec2f,
  @builtin(instance_index) instance: u32,
};

struct VertexOutput {
  @builtin(position) pos: vec4f,
};

@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(input: VertexInput) -> VertexOutput  {
  let i = f32(input.instance);
  let cell = vec2f(i % grid.x, floor(i / grid.x));
  let cellOffset = cell / grid * 2;
  let gridPos = (input.pos + 1) / grid - 1 + cellOffset;
  
  var output: VertexOutput;
  output.pos = vec4f(gridPos, 0, 1);
  return output;
}
```

Notice that this requires you to refer to the input position and instance index with `input`, and the struct that you return first needs to be declared as a variable and have its individual properties set. In this case, it doesn't make too much difference, and in fact makes the shader function a bit longer, but as your shaders grow more complex, using structs can be a great way to help organize your data.

<a id="pass-data-between-the-vertex-and-fragment-functions"></a>

## **Pass data between the vertex and fragment functions**

As a reminder, your `@fragment` function is as simple as possible:

<a id="index.html-createshadermodule-call_7"></a>

### index.html (createShaderModule call)

```
@fragment
fn fragmentMain() -> @location(0) vec4f {
  return vec4f(1, 0, 0, 1);
}
```

You are not taking any inputs, and you are passing out a solid color (red) as your output. If the shader knew more about the geometry that it's coloring, though, you could use that extra data to make things a bit more interesting. For instance, what if you want to change the color of each square based on its cell coordinate? The `@vertex` stage knows which cell is being rendered; you just need to pass it along to the `@fragment` stage.

To pass any data between the vertex and fragment stages, you need to include it in an output struct with a `@location` of our choice. Since you want to pass the cell coordinate, add it to the `VertexOutput` struct from earlier, and then set it in the `@vertex` function before you return.

1. Change the return value of your vertex shader, like this:

<a id="index.html-createshadermodule-call_8"></a>

### index.html (createShaderModule call)

```
struct VertexInput {
  @location(0) pos: vec2f,
  @builtin(instance_index) instance: u32,
};

struct VertexOutput {
  @builtin(position) pos: vec4f,
  @location(0) cell: vec2f, // New line!
};

@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertexMain(input: VertexInput) -> VertexOutput  {
  let i = f32(input.instance);
  let cell = vec2f(i % grid.x, floor(i / grid.x));
  let cellOffset = cell / grid * 2;
  let gridPos = (input.pos + 1) / grid - 1 + cellOffset;
  
  var output: VertexOutput;
  output.pos = vec4f(gridPos, 0, 1);
  output.cell = cell; // New line!
  return output;
}
```

2. In the `@fragment` function, receive the value by adding an argument with the same `@location`. (The names don't have to match, but it's easier to keep track of things if they do!)

<a id="index.html-createshadermodule-call_9"></a>

### index.html (createShaderModule call)

```
@fragment
fn fragmentMain(@location(0) cell: vec2f) -> @location(0) vec4f {
  // Remember, fragment return values are (Red, Green, Blue, Alpha)
  // and since cell is a 2D vector, this is equivalent to:
  // (Red = cell.x, Green = cell.y, Blue = 0, Alpha = 1)
  return vec4f(cell, 0, 1);
}
```

3. Alternatively, you could use a struct instead:

<a id="index.html-createshadermodule-call_10"></a>

### index.html (createShaderModule call)

```
struct FragInput {
  @location(0) cell: vec2f,
};

@fragment
fn fragmentMain(input: FragInput) -> @location(0) vec4f {
  return vec4f(input.cell, 0, 1);
}
```

4. Another alternative, since in your code both of these functions are defined in the same shader module, is to reuse the `@vertex` stage's output struct! This makes passing values easy because the names and locations are naturally consistent.

<a id="index.html-createshadermodule-call_11"></a>

### index.html (createShaderModule call)

```
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
  return vec4f(input.cell, 0, 1);
}
```

No matter which pattern you chose, the result is that you have access to the cell number in the `@fragment` function and are able to use it in order to influence the color. With any of the above code, the output looks like this:

![A grid of squares where the leftmost column is green, the bottom row is red, and all other squares are yellow.](<your-first-webgpu-app/ef4ff8ca1b3c974e6539fe7d.png>)

There are definitely more colors now, but it's not exactly nice looking. You might wonder why only the left and bottom rows are different. That's because the color values that you return from the `@fragment` function expect each channel to be in the range of 0 to 1, and any values outside of that range are clamped to it. Your cell values, on the other hand, range from 0 to 32 along each axis. So what you see here is that the first row and column immediately hit that full 1 value on either the red or green color channel, and every cell after that clamps to the same value.

If you want a smoother transition between colors, you need to return a fractional value for each color channel, ideally starting at zero and ending at one along each axis, which means yet another divide by `grid`!

5. Change the fragment shader, like this:

<a id="index.html-createshadermodule-call_12"></a>

### index.html (createShaderModule call)

```
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
  return vec4f(input.cell/grid, 0, 1);
}
```

Refresh the page, and you can see that the new code *does* give you a much nicer gradient of colors across the entire grid.

![A grid of squares that transition from black, to red, to green, to yellow in different corners.](<your-first-webgpu-app/064f609e2b95d988ba6113a8.png>)

While that's certainly an improvement, there's now an unfortunate dark corner in the lower left, where the grid becomes black. When you start doing the Game of Life simulation, a hard-to-see section of the grid will obscure what's going on. It would be nice to brighten that up.

Fortunately, you have a whole unused color channel—blue—that you can use. The effect that you ideally want is to have the blue be brightest where the other colors are darkest, and then fade out as the other colors grow in intensity. The easiest way to do that is to have the channel *start* at 1 and subtract one of the cell values. It can be either `c.x` or `c.y`. Try both, and then pick the one you prefer!

6. Add brighter colors to the fragment shader, like this:

<a id="createshadermodule-call"></a>

### createShaderModule call

```
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
  let c = input.cell / grid;
  return vec4f(c, 1-c.x, 1);
}
```

The result looks quite nice!

![A grid of squares that transition from red, to green, to blue to yellow in different corners.](<your-first-webgpu-app/956edd9552fa2daeec5c9cf6.png>)

This isn't a critical step! But because it looks better, it's included it in the corresponding checkpoint source file, and the rest of the screenshots in this codelab reflect this more colorful grid.

<a id="7-manage-cell-state"></a>

Source: [https://codelabs.developers.google.com/your-first-webgpu-app#5](<https://codelabs.developers.google.com/your-first-webgpu-app#5>)
