# 2. The Graphics Rendering Pipeline

The graphics rendering pipeline (a.k.a. the pipeline) is a sequence of stages that run in parallel and are dependent on the previous stage. This means that the pipeline is as slow as its slowest stage.

There are 4 big stages in the graphics pipeline, each one divided in many smaller steps:

- Application
- Geometry Processing
- Rasterization
- Pixel Processing

## 2.2 Application Stage

The Application stage is the CPU work. It can do previous calculations like collision detection or input handling. It prepares the geometry data for the GPU.

## 2.3 Geometry Processing Stage

The Geometry Processing stage is divided into the following stages

- Vertex Shading
- Projection
- Clipping
- Screen Mapping

### 2.3.1 Vertex Shading and Projection

The vertex shading historically used to compute color per vertex (Gouraud shading). Now the name stayed but in reality it performs transformations on the vertex positions to clip space (model → view → projection). Moreover, the Vertex Shader can output arbitrary data for the Pixel Shader to get as input and use in its computing (Normals, Tangents, Texture Coordinates). These outputs are interpolated across the primitive before reaching the Pixel Shader, which is what makes per-vertex data usable per-pixel.

### 2.3.2 Optional Vertex processing

Depending on the HW, the output of the Vertex Shading could be processed by one or more of the following operations:

#### Tessellation

The tessellation generates vertices from the original set, the amount depending for example in how close is the object to the camera. The idea is to have a smoother surface (more vertices) when the object is close and a rougher surface (less vertices) when the object is far away. There are 3 sub-divisions of this process: **Hull Shader**, **Tessellator** and **Domain Shader**.

#### Geometry Shader

Older than tessellation, it's more common across GPUs. It consists in a simpler vertex generation, for example for particle generation. It can generate a quad from a point, making it easier to shade.

#### Stream Output

This is a more general step, the vertex output is taken as an array that can be processed by both CPU or GPU, typically used for particle simulations.

### 2.3.3 Clipping

Before passing the data to the Rasterization stage, we must be sure to only render what's visible, i.e. what's partially or totally inside the *view volume*. That's why it's so important to transform the vertex coordinates into the homogeneous space, so everything it's compared to the unit cube limits.

If a primitive is totally inside the unit cube, it's rasterized as it is. If it's totally outside, it's ignored. If it's partially inside, the outer vertex must be replaced with new vertices in the intersection of the primitive and the unit cube.

The user may also define extra clipping spaces. This is called **sectioning**. Finally the clipped vertices are transformed with a **perspective division** so they fit a *normalized device coordinates*.

### 2.3.4 Screen Mapping

After clipping, the vertex coordinates are transformed to **screen coordinates**. If *x* and *y* joined with the *z*-coordinate, they make the **window coordinates**.

The mapping is done by a translation and a scale. The *x* and *y* coordinates fit the screen and the *z* component is mapped to a per-API arbitrary range. When all the transformations are done, these coordinates are passed on to the rasterizer stage.

As a side comment, the screen coordinates are floating point values even though we have an integer number for the amount of pixels per row and column. This means that the center of the pixel is calculated as its index plus 0.5.

## 2.4 Rasterization

Also known as *scan conversion*, rasterization is the process of finding all the pixels within a primitive. It's the conversion from window coordinates (screen + z) to pixels on the screen. The rasterization has two steps: **triangle setup/primitive assembly** and **triangle traversal**.

The interesting part of rasterization is that one must define what it means for the pixel to be "inside" the primitive. Some of the heuristics are:

- Single sampling: if the center of the pixel is inside the primitive, then the whole pixel is.
- Supersampling/Multisampling antialiasing: same base idea but with more samples per pixel.
- Conservative rasterization: a pixel is inside the primitive if some part (no matter how much) of the pixel overlaps.

### 2.4.1 Triangle Setup

Fixed-function HW is used here to perform differential/edge equations. This can be used later in triangle traversal or maybe for interpolation of the shading data produced by the geometry processing stage.

### 2.4.2 Triangle Traversal

This is the moment where the sampling is performed and the **fragments** are created, interpolating shading data from the original vertices. There's also a *perspective-correct interpolation* (more on that later).

## 2.5 Pixel Processing

### 2.5.1 Pixel Shading

The Pixel Shading consists of processing the interpolated shading data to output one or more colors per fragment for the next step. While the rasterization is done by dedicated hardwired silicon, the pixel shading is executed in the GPU cores and it's programmable (what we call a pixel/fragment shader program). Many techniques are performed here such as **texturing** and **illumination computing**.

### 2.5.2 Merging

The merging step receives a color buffer as the input and must perform operations with this buffer and the current color buffer to see what's the next result for each pixel. Although this step is not programmable, it's highly configurable and it's called ROP (Raster operations pipeline/Render output unit).

One of the algorithms performed here is the **Depth/*z*-buffer comparison**. The GPU compares primitives *z* value and keeps the lesser one because it means it's closer to the viewer. The good thing is that this is independent of the order, but it doesn't work out of the box for partially transparent primitives. They must be rendered in order after the opaques or just use another algorithm.

Modern GPUs also implement **Early-Z** (or **Hi-Z**): the depth test is performed *before* the pixel shader runs, discarding fragments that will be occluded without ever shading them. This is a major optimization. It can be defeated if the pixel shader writes to depth or uses `discard`.

The **alpha channel** is associated with the color buffer and can be used in *alpha-testing* to discard or even merge pixel colors. This is a way to ensure that transparent primitives do not affect the *z*-buffer.

A **stencil buffer** can be used to filter the rendering, creating sections of rendering or no-rendering.

All in all, the **Framebuffer** contains all these buffers and all the operations at the end of the pipeline are called *raster operations* or *blend operations*. When the pipeline finishes its last stage, the frame buffer is presented as the *front buffer* on screen. To prevent the viewer from seeing how the pixels are re-drawn every frame, a *back buffer* is always being written while the front one is being read. This and similar techniques are called **double buffering** (or **triple buffering** when a third buffer is added to reduce stutter).

# 3. The Graphics Processing Unit

## 3.1 Data-Parallel Architectures

The GPU has **shader cores** that have to compute the same program, so the instructions will be the same. That's why the **SIMT** (Single Instruction, Multiple Threads) architecture fits perfectly. The distinction from classical CPU SIMD is subtle: SIMD means one core operates on a vector register simultaneously; SIMT means many threads each with their own registers execute the same instruction in lockstep across a warp. GPUs use SIMD lanes internally, but the abstraction exposed to shader programs is SIMT. For the same instruction, we can process multiple threads of data (e.g. fragment shading). 

Each program execution is called **thread** and they group into **warps/wavefronts**. This is useful because even though the arithmetic operations are fast enough, when the threads have to fetch data from memory, the whole core is stalled. Instead of waiting, the GPU switches context to another warp and starts over with the following threads. By the time all the warps are stalled, ideally the first warp will be ready to continue execution after fetching the data from memory. This technique is called **latency hiding**.

To ensure parallelism, I deduce that avoiding conditional execution with whiles and ifs is a **must**. The precise term for the problem is **thread divergence** (or branch divergence): when threads in the same warp take different branches, the warp executes **both paths serially**, masking inactive threads. So `if`/`while` aren't forbidden — they're only costly when threads in the *same warp* diverge. Uniform branches (all threads take the same path) are essentially free. **Rule of thumb**: branches on **uniforms** or **spatially coherent data** (e.g. screen position, since GPUs pack nearby pixels into the same warp) are cheap; branches on noisy per-fragment data (texture samples, random values) are expensive. Apart from that, there seems to be a sweet spot between instructions and *occupancy* (amount of warps). Also the number of threads depends on the registers they need. The more registers they need, the fewer threads can be created, resulting in fewer warps. Occupancy is also limited by **shared memory** usage per block, not just registers.

## 3.2 GPU Pipeline Overview

### Logical Model

1. Vertex Shader              -> programmable, mandatory.
2. Tessellation               -> programmable, optional.
3. Geometry Shader            -> programmable, optional.
4. Clipping                   -> fixed, mandatory.
5. Screen Mapping             -> configurable, mandatory.
6. Triangle Setup & Traversal -> fixed, mandatory.
7. Pixel Shader               -> programmable, mandatory.
8. Merger                     -> configurable, mandatory.

The logical model must not be mixed with the actual *physical model*, i.e. the implementation of the pipeline by the GPU vendors.

**Clipping is listed as "fixed" but earlier it said the user can define extra clipping planes — how does this work?** "Fixed" means non-programmable, not non-configurable. User-defined clipping planes are passed as uniform parameters. The vertex shader writes a signed distance per custom plane into a special output (`gl_ClipDistance` in GLSL, `SV_ClipDistance` in HLSL), and the fixed-function clipper hardware discards or clips geometry where that value is negative. The clipping *mechanism* is hardwired; the clipping *planes* are configurable.

Note that both Tessellation and Geometry Shader aren't supported in all GPUs, especially mobile devices.

## 3.3 The Programmable Shader Stage

Nowadays, there is a unified shader design for most of the shader programs, i.e. they have the same *instruction set architecture*. A processor implementing this model is called a **common-shader** in DirectX and the GPU would have a **Unified Shader Architecture**.

The shader languages usually have C-Style syntax and the most common are DirectX *High-Level Shading Language* (HLSL) and *OpenGL Shading Language* (GLSL). Even more, the HLSL shaders can be compiled to VM bytecode or *intermediate language*.
Shaders can be compiled offline of the main rendering program.
The vertex shading historically used to compute color per vertex. Now the name stayed but in reality it performs transformations on the vertex positions given a model, a view and a projection. Moreover, the Vertex Shader can output arbitrary data for the Pixel Shader to get as input and use in its computing (Normals, Tangents, Texture Coordinates). 

The basic data types are 32-bit single-precision floating points, representing scalars and vectors (in the language, the hw doesn't support vectors actually). Modern GPUs support 32-bit integers and 64-bit floats. Aggregated data (structures, arrays) is also supported.

A draw call invokes the graphics API to execute a shader program. All programs take two kinds of inputs:

- Uniforms: constant along the whole draw call.
- Varying: data that come from vertex info or from rasterization.
- Texture: is something like a uniform but nowadays it can be thought as any large array of data.

There's a VM that provides registers according to the inputs and outputs data types. Register types are: **constant registers** (for uniforms, read-only), **temporary registers** (scratch space — intermediate storage for mid-shader calculations, like temp variables in a function), **input registers** (varying data coming in: vertex attributes or interpolated fragment values), and **output registers** (results written out, e.g. final color or transformed position). So varying input/output has its own register category, separate from constants and temporaries.

The shader languages provide operators and functions optimized for the GPU and there's also flow control. It's static when applied to uniforms and it can be optimized, but if it's dynamic due to applying to varying input, it can be costly if it causes thread divergence.

## 3.4 The Evolution of Programmable Shading and APIs

### The First Steps

- In 1984 Cook's *shade trees* define a tree with a concatenation of operations for a simple shader.
- In the late 80's the RenderMan Shading Language was developed, still used today in film rendering and is the base for the *Open Shading Language* project.
- The first consumer-level graphics hw *Voodoo* was released in 1996 by 3dfx Interactive. It was quickly adopted mainly for its performance with Quake.
- The *Quake III: Arena* scripting language was an early attempt to implement programmable shading through render passes with some success in 1999.
- NVIDIA's GeForce 256 was the first so-called GPU (NVIDIA coined the term with this product in 1999). It wasn't programmable but highly configurable.
- NVIDIA's GeForce 3 was the first GPU with support for programmable vertex shaders using DirectX 8.0 or OpenGL with extensions. It had some support for pixel shading but it was limited to some operations and only 12 instructions.

### The Break with DirectX 9 and its Evolution with OpenGL

In 2002 DirectX 9.0 brought some major changes. Previously there was no branching so both options had to be calculated and then interpolated. Also DirectX defined a **Shader Model** (SM) to distinguish hw with different capabilities (SM 2.0 with DirectX 9.0). Even more, they added float 16 and dependent texture reads support so finally the pixel shading had all the resources needed by the time.

As the assembly code was getting more resourceful and complex, DirectX 9 also presented HLSL in collaboration with NVIDIA. Around the same time OpenGL released GLSL. Both languages have C-style syntax and are based in RenderMan Shading Language features.

SM 3.0 was introduced in 2004 and brought texture reading in vertex shaders and upgraded many optional features to requirements. The PS3's RSX (designed by NVIDIA) was a traditional SM3.0 part. The Xbox 360's Xenos GPU (designed by ATI) was from the same era but featured a **unified shader architecture** — blurring the vertex/pixel shader distinction and anticipating SM4.0 concepts. It's worth noticing that the Nintendo Wii still used a fixed pipeline while at the same time there were already tools for visual shader programming resembling Cook's shade tree.

In late 2006 DirectX 10 was launched with SM 4.0, bringing uniforms, geometry shaders, streamed output, integer data type and bitwise operations. A similar model was presented by OpenGL with GLSL 3.30.

With DirectX 11 in 2009, SM 5.0 was presented with tessellation, compute shaders and support for CPU multiprocessing. OpenGL supported those features a little bit later with GLSL 4.30.

It's worth noticing that the main advantage in time from DirectX in comparison to OpenGL is that it's developed by Microsoft in collaboration with a few vendors (NVIDIA, AMD and Intel), while OpenGL is developed by non-profit Khronos Group with a much wider vendor collaboration. This implies that the features must work in a bigger range of architectures and that's why OpenGL provides support for extensions prior to bringing official support.

### Modern APIs

In 2013 AMD released the Mantle API in collaboration with DICE. They introduced a **low-overhead** model: traditional APIs (OpenGL, DX11) do a lot of implicit work on the CPU per draw call — state validation, resource synchronization, on-the-fly shader compilation. Low-overhead means the API shifts that responsibility to the developer: explicit memory management, pre-compiled pipeline state objects, no implicit synchronization, and multi-threaded command recording. This dramatically reduces CPU cost per draw call, which was the bottleneck Mantle was targeting. Microsoft took this model and released DirectX 12 in 2015. They didn't add hw features from their last version but refactored the whole API to the same Mantle new standard. Later on, AMD donated Mantle to Khronos Group and in 2016 they released Vulkan. While the idea was the same, Vulkan stands out as it works in many operating systems and devices (PCs, mobile) and can work without a display window (e.g. computing).

### Mobile API

As the mobile hw wasn't working well with the OpenGL API, they released OpenGL ES 1.0 in 2003 with a fixed-function pipeline. Following versions added programmable shaders and even compute shaders or tessellation. As a consequence of this development, the browser-based API WebGL was released in 2011 and with JavaScript calls, OpenGL ES 2.0 was ported to the web and could be used in almost any device. Later on, WebGL 2.0 would be released based in OpenGL ES 3.0.

## 3.5 The Vertex Shader

While this is the first programmable stage of the pipeline, actually there's a previous stage called **input assembly** where the vertex input is organized in some specific way for the shader and it also can perform *instancing*.

The Vertex data may contain more than position data, mostly normals, colors, uv coordinates. The only thing mandatory for the vertex shader is to output a position for the rasterization stage (or some previous optional stage). Most of the times, the position will be in the vertex input and it will be transformed to the output space, the *Clip space*. 

**Model -> World -> View -> Clip**

## 3.6 The Tesselation Stage

This is an optional stage at the pipeline level but a mandatory hardware capability from DX11 onwards. Any DX11-class GPU must implement the three sub-stages in hardware, but you choose whether to bind hull/domain shaders per draw call — if you don't, the stage is skipped entirely. It allows us to render curved surfaces. Their descriptions often require less data than triangles so more memory is saved and the bus between CPU and GPU is lighter, reducing a bottleneck risk.

The **Level of Detail (LOD)** can be managed here, creating a variable amount of triangles depending on the distance of the object to the camera or the GPU capabilities.

The tesselation stage has 3 sub-stages:

- **Hull shader/Tesselation control shader**: has an input patch and defines control points for the domain shader.
- **Tesselator/Primitive generator**: fixed functin that generates points based on the control points, tesselation factors and a type.
- **Domain shader/Tesselation evaluation shader**: takes the barycentric coordinates generated for each point and assigns the vertex data for each one (normal, uv coordinates, color, etc).

## 3.7 The Geometry Shader

This stage turns primitives into other primitives (e.g. can transform triangles in line edges to get a wireframe view). Although it can elaborate **patches** (a higher-order primitive type representing a set of control points — 1–32 in DX11 — that describe a curved surface such as Bézier or B-spline), the tessellation stage is better for this task because it has dedicated fixed-function hardware for subdivision. The geometry shader is mainly used to modify incoming data, for example to simultaneously render the six faces of a cubemap or to create cascaded shadow maps efficiently. 

Some other facts:

- It also can perform instancing.
- It can output up to four **streams**: separate output buffers the GS writes primitive vertex data to simultaneously. Each stream is a GPU buffer (bound as a vertex buffer) that can be consumed by a subsequent draw call or read back to the CPU — the mechanism that enables multi-pass simulations staying entirely on the GPU. Only stream 0 feeds the rasterizer; streams 1–3 are stream-output only.
- It guarantees order preservation (which goes agains GPU parallelism).

## 3.7.1 Stream Output

The *Stream ouput/Transform feedback* is an option pipeline tool that allow the primitive vertex data to be reused in the pipeline, whether it's sent in parallel to the rasterization stage or this one is totally disabled. This is mainly used for simulations or skin a model and reuse the vertices. It preserves primitives order and early DX10 stream output was limited to float arrays, but DX11+ and Vulkan support structured buffers with arbitrary element types (integers, structs, etc.). It is still expensive memory-wise due to the order preservation requirement.

## 3.8 The Pixel Shader

After the rasterizer samples the pixels to see if they overlap with the primitives, the ones that do are called **fragments** and become the input for the Pixel Shader. For these fragments, the values from the vertex shader output are interpolated and the interpolation technique is configurable (mainly perspective-correct but might be other one such as screen-space interpolation). 

Some things the pixel shader can do:

- Alpha testing
- Read/write depth buffer
- Read/write stencil buffer (at least in modern APIs)
- **Discard fragments**: this is specially important. For exmaple it might use a user defined clip space.

In modern APIs, the pixel shader can write its output to **Multiple Render Targets** instead of just writing a color an z-buffer. This has given rise to another type of pipeline, the **Deferred rendering** pipeline, where a first render pass writes the geometry and material data and then a second pass performs the lighting calculations.

Another important limitation of the pixel shader is that it cannot access neighbour fragments data. However, there are exceptions for this, maybe the more relevant one being the capacity of querying the gradient data of the *quad neighbours*. As a side note, this gradient data cannot be obtained inside a code branch as it requires parallelisim and synchronization between the shader cores processing these quad neighbours.

There are some special mechanisms that allow different shader cores to read/write the same buffer. One is **Unordered acces view (UAV)/Shader storage buffer object (SSBO)** and the second one is the **Rasterizer order views (ROV)**. The former is just a read/write memory chunk, accessed from any shader. This obviously leads to data races and it doesn't guarantee order of execution. The latter does guarantee this, but at the cost of a stall if an out of order execution is detected.

## 3.9 The Merging Stage

The **output merger/per-sample operations** stage gathers the pixel shader output and performs depth testing, stencil testing and blending. Sometimes the GPU can perform *early-z test* to avoid computing the pixel shader for a fragment that won't be visible in the end. However, this is disabled by default if the pixel shader modifies the z value of discards the frament (though it can be enforced). The merging stage is hightly configurable, mainly for the color blending. It's able to perform many algorithms for this operation.

## 3.10 The Compute Shader

In parallel to the graphics pipeline there is another use for the GPU, i.e. **computing**. The compute shader can access the same buffers as the other shaders (mainly for textures) and it's more explicitly related to the GPU harwdware in that the memory is shared between a **thread group** that is guaranteed to run concurrently.

Another advantage of using compute shaders is that it can take as input data that's already located on the GPU so it's much faster that sending it to the CPU for the calculations. This is why the compute shader is used for post-processing of the graphics pipeline data as well as particle systems, mesh processing, culling, etc.

# 4. Transforms
