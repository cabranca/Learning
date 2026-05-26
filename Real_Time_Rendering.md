# 2. The Graphics Rendering Pipeline

The graphics rendering pipeline (a.k.a. the pipeline) is a sequence of stages that run in parallel and are dependant of the previous stage. This means that the pipeline is as slow as its slowest stage.

There are 4 big stages in the graphics pipeline, each one divided in many smaller steps:

- Application
- Geometry Processing
- Rasterization
- Pixel Processing

## 2.2 Application Stage

The Application stage is the CPU work. It can do previous calculations like collision detection or input handling. It prepares the geometry data for the GPU.

## 2.3 Geometry Processing Stage

The Geometry Processing stageis divided into the following stages

- Vertex Shading
- Projection
- Clipping
- Screen Mapping

### 2.3.1 Vertex Shading and Projection

The vertex shading historically used to compute color per vertex (Gouraud shading). Now the name stayed but in reality it performs transformations on the vertex positions to clip space (model → view → projection). Moreover, the Vertex Shader can output arbitrary data for the Pixel Shader to get as input and use in its computing (Normals, Tangents, Texture Coordinates). These outputs are interpolated across the primitive before reaching the Pixel Shader, which is what makes per-vertex data usable per-pixel.

### 2.3.2 Optional Vertex processing

Depending on the HW, the output of the Vertex Shading could be processed by one or more of the following operations:

#### Tesellation

The tesellation generates vertices from the original set, the amount depending for example in how close is the object to the camera. The idea is to have a smoother surface (more vertices) when the object is close and a rougher surface (less vertices) when the object is far away. There are 3 sub-divisions of this process: **Hull Shader**, **Tesellator** and **Domain Shader**.

#### Geometry Shader

Older that tesellation, it's more common across GPUs. It consist in a simpler vertex generation, for example for particle generation. It can generate a quad from a point, making it easier to shade.

#### Stream Output

This is a more general step, the vertex output is taken as an array that can be processed by both CPU or GPU, tipically used for particle simulations.

### 2.3.3 Clipping

Before passing the data to the Rasterization stage, we must be sure to only render what's visible, i.e. what's partially or totally inside the *view volume*. That's why it's so important to transform the vertex coordinates into the homogeneous space, so everything it's compared to the unit cube limits.

If a primitive is totally inside the unit cube, it's rasterized as it is. If it's totally outside, it's ignored. If it's partially inside, the outer vertex must be replaced with new vertices in the intersection of the primitive and the unit cube.

The user may also define extra clipping spaces. this is called **sectioning**. Finally the clipped vertices are transformed with a **perspective division** so they fit a *normalized device coordinates*.

### 2.3.4 Screen Mapping

After clipping, the vertex coordinates are transformed to **screen coordinates**. If *x* and *y* joined with the *z*-coordinate, they make the **window coordinates**.

The mapping is done by a translation and a scale. The *x* and *y* coordinates fits the screen and the *z* component is mapped to an per-API arbitrary range. When all the transformations are done, these coordinates are passed on to the rasterizer stage.

As a side comment, the screen coordinate are floating point values even though we have an integer number for the amount of pixels per row and column. This means that the center of the pixel is calculated as its index plus 0.5.

## 2.4 Rasterization

Also known as *scan conversion*, rasterization is the process of finding all the pixels within a primitive. It's the conversion from window coordinates (screen + z) to pixels on the screen. The rasterization has two steps: **triangle setup/primitive assembly** and **triangle traversal**.

The interesting part of rasterization is that one must define what it means for the pixel to be "inside" the primitive. Some of the heuristics are:

- Single sampling: if the center of the pixel is inside the primitive, then the whole pixel is.
- Supersampling/Multisampling antialiasing: same base idea but with more samples per pixel.
- Conservative rasterization: a pixel is inside the primitive if some part (no matter how much) of the pixel overlaps.

### 2.4.1 Triangle Setup

Fixed-function HW is used here to perforom differential/edge equations. This can be used later in triangle traversal or mabye for interpolation of the shading data produced by the geometry processing stage.

### 2.4.2 Triangle Traversal

This is the moment where the sampling is performed and the **fragments** are created, interpolating shading data from the original vertices. Tehre's also a *perspective-correct interpolation* (more on that later).

## 2.5 Pixel Processing

### 2.5.1 Pixel Shading

The Pixel Shading consists of processing the interpolated shading data to output one or more colors per fragment for the next step. While the rasterization is done by dedicated hardwired silicon, the pixel shading is executed in the GPU cores and it's programmable (what we call a pixel/fragment shader program). Many techniques are performed here such as **texturing** and **illuminations computing**.

### 2.5.2 Merging

The merging step receives a color buffer as the input and must perform operations with this buffer and the current color buffer to see what's the next result for each pixel. Althought this step is not programmable, it's highly configurable and it's called ROP (Raster oprations pipeline/Render output unit).

One of the algorithms performed here is the **Depth/*z*-buffer comparisson**. The GPU compares primitives *z* value and keeps the lesser one because it means it's closer to the viewer. The good thing is that this is independent of the order, but it doesn't work out of the box for partially transparent primitives. They must be rendered in order after the opaques or just use another algorithm.

The **alpha channel** is asocciated with the color buffer and can be used in *alpha-testing* to discard or even merge pixel colors. This is a way to ensure that transparent primitives do not affect the *z*-buffer.

A **stencil buffer** can be used to filter the rendering, creating sections of rendering or no-rendering.

All in all, the **Framebuffer** contains all this buffers and all the operations at the end of the pipeline are called *raster operations* or *blend operations*. When the pipeline finishes its last stage, the frame buffer is presented as the *front buffer* on screen. To prevent the viewer from seeing how the pixels are re-drawn every frame, a *back buffer* is always being written while the front when is being read. This one of many techniques is called **double buffering**.

# 3. The Graphics Processing Unit

## 3.1 Data-Parallel Architectures

The GPU has **shader cores** that have to compute the same program, so the instructions will be the same. That's why the SIMD architecture fits perfectly. For the same instructions, we can process multiple data (e.g. fragment shading). 

Each program execution is called **thread** and they group into **warps/wavefronts**. This is useful because even though the arithmetic operations are fast enough, when the threads have to fetch data from memory, the whole core is stalled. Instead of waiting, the GPU switches context to another warp and starts over with the following threads. By the time all the warps are stalled, ideally the first warp will be ready to continue execution after fetching the data from memory. This technique is called **latency hiding**.

To ensure parallelism, I deduce that avoiding conditional execution with whiles and ifs is a **must**. The precise term for the problem is **thread divergence** (or branch divergence): when threads in the same warp take different branches, the warp executes **both paths serially**, masking inactive threads. So `if`/`while` aren't forbidden — they're only costly when threads in the *same warp* diverge. Uniform branches (all threads take the same path) are essentially free. **Rule of thumb**: branches on **uniforms** or **spatially coherent data** (e.g. screen position, since GPUs pack nearby pixels into the same warp) are cheap; branches on noisy per-fragment data (texture samples, random values) are expensive. Apart from that, there seems to be a sweet spot between instructions and *occupancy* (amount of warps). Also the amount threads are dependant on the registers they need. The more registers they need, the less threads can be created, resulting in fewer warps. Occupancy is also limited by **shared memory** usage per block, not just registers.

## 3.2 GPU Pipeline Overview

### Logical Model

1. Vertex Shader              -> programmable, mandatory.
2. Tesselation                -> programmable, optional.
3. Geometry Shader            -> programmable, optional.
4. Clipping                   -> fixed, mandatory.
5. Screen Mapping             -> configurable, mandatory.
6. Triangle Setup & Traversal -> fixed, mandatory.
7. Pixel Shader               -> programmable, mandatory.
8. Merger                     -> configurable, mandatory.

The logical model must not be mixed with the actual *physical model*, i.e. the implementation of the pipeline by the GPU vendors.

Here it says that the clipping is fixed, but before it said that clipping planes may be added by the user, so how does this work?

Note that both Tesselation and Geometry Shader aren't supported in all GPUs, especially mobile devices.

## 3.3 The Programmable Shader Stage

Nowadays, there is a unified shader design for most of the shader programs, i.e. they have the same *isntruction set architecture*. A processor implementing this model is called a **common-shader** in DirectX and the GPU would have a **Unified Shader Architecture**.

The shader languages usually have C-Style syntax and the most common are DirectX *High-Level Shading Language* (HLSL) and *OpenGL Shading Language* (GLSL). Even more, the HSLS shaders can be compiled to VM bytecode or *intermediate language*.
Shaders can be compiled offline of the main rendering program.
The vertex shading historically used to compute color con vertex. Now the name stayed but in reality it performs transformations on the vertex positions given a model, a view and a projection. Moreover, the Vertex Shader can output arbitrary data for the Pixel Shader to get as input and use in its computing (Normals, Tangents, Texture Coordinates). 

The basic data type are 32-bit single-precision floating points, representing scalars and vectors (in the language, the hw doesn't support vectors actually). Modern GPUs suppor 32-bit integers and 64-bit floats. Aggregated data (structures, arrays) is also supported.

A draw call invokes the graphics API to execute a shader program. All programs take two kind of inputs:

- Uniforms: constant along the whole draw call.
- Varying: data that come from vertex info or from rasterization.
- Texture: is something like a uniform but nowadays it can be thought as any large array of data.

There's a VM that provide registers acording to the inputs and outputs data types. There are many constant registers for the uniforms and a fw temporary registars used for stractch space (what does this mean? Are any of these two types for varying input/output or is there a third kind of register?).

The shader languages provide operators and functions optimized for the GPU and there's also flow control. It's statically when applied to uniforms and it can be optimized, but if it's dinamically due to applying to varying input, it can be costly if it causes thread divergence.

## 3.4 The Evolution of Programmable Shading and APIs
