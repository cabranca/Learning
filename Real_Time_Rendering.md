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

# 5. Shading Basics

## 5.2 Light Sources

The light source gives some sort of a dominant direction for shading (the angle between view and light or some variation of this). There are shading models that try to follow physic laws and some others that seek a specific artistic style.

At a high level, we'll always want two terms, one for the unlit contribution and one for the lit contributions, i.e. the sum of two sub-functions. The Unlit one would represent the color of the fragment in de dark and the lit one should be related to the color of the light source as well.

```color_shaded = f_unlit(n,v) + color_light * f_lit(l, n, v)``` (or a summatory for multiple light sources)

We also see a proportionality between the contribution of the light and the dot product between light and normal vectors when positive. We have to clamp the value to zero from below because negative values indicate that the light comes from inside the object, something we don't wanna consider. Even though many models consider this proportion, there are some of them that do not, so they don't perform this dot product. Examples: **wrap shading** (used to approximate subsurface scattering in skin/cloth) replaces the clamped n·l with (n·l + w)/(1+w), effectively wrapping light around the terminator; **anisotropic BRDFs** (for brushed metal or hair) substitute or supplement the normal with a tangent vector, producing a fundamentally different geometric term.

Maybe the simplest choice for the lit function is just taking a constant color. This is the **Lambertian** shading model — and yes, this is exactly the diffuse term of Phong. The Lambertian BRDF is a constant (k_d, the surface color), meaning the surface reflects light equally in all directions. The n·l factor comes from the geometry of light incidence, not from the BRDF itself.

```c_shaded = f_unlit(n,v) + clamp(dot(l,n), 0, 1) * c_light * c_surface ```

### 5.2.1 Directional Light

This is the simplest model, it's an abstraction of a light source that is so far away that the light rays become parallel (like the sun at human scale). It has a constant direction and color. However, for creativity reasons, the color might not be constant across the whole space.

### 5.2.2 Punctual Lights

This type of lights have a position, an origin for the source. In this case, the light vector is dependent of the fragment position relative to the light source.

One example is the **Point/Omni light** that irradiates light in every direction and it decrases at a square distance ratio. Dividing by the squared distance has two big problems. Firstly, at the origin there is a singularity so one must use some alternatives that approximate this calculation, mainly adding an epsilon to the distance term or clamping it to a minimum distance. The latter has a physical explanation: it considers the light source actual radius, so any smaller distance value would represent a point inside the source which doesn't make sense. Secondly, at hight distances the numbers become more and more costly but the color value is practically zero. To avoid this, an option could be to multiply a **windowing function** by the main function in order to set the color to zero smoothly or to change the color function to a linear decrase after some distance threshold. It depends on the application requirements. All in all, there might be another **distance falloff functions** in engines that have other requirements such as performance or style choices.

Another punctual light example is the **Spotlight**. In addition to the distance falloff function there is a function that decreases the light intensity depending on the light vector. The most common spotlight model has a **penumbra** angle that defines a cone where the light is at the maximum intensity (max value of the directional falloff function) and a **umbra** angle that defines the limit of the light cone. Any fragment outside of the cone is not lighted.

### 5.2.3 Other Light Types

Although the directional and punctual lights are the mainstream, there's been an increase in **area lights**, a model closer to the real world where lights have a size to consider. They aren't that expensive and the advance in GPU computing will continue to make them more likeable to use.

# 6. Texturing

## 6.1 The Texturing Pipeline

### 6.1.1 The Projector Function

Basically we have multiple ways to transform a coordinate in **Object Space** (e.g. a vertex in a mesh) to a coordinate in **Parameter Space**. We have different types of projections (planar, spheric, 2D to 4D, based on position, temperature or whatever we want). Maybe the most common one is to bake the *(u,v)* coordinates of a 2D texture to the vertices of a mesh when authoring a model.

### 6.1.2 The Corresponder Function

This function takes the (*u*, *v*) coordinates in parameter space and transform them to **Texture Space**. The texture space has the size of the actual texture with some width and height for their **texels**. As there is a mapping between this range and the parameter space range [0, 1], the user must define what happens outside of this range. The common approaches are:

- Wrap/Repeat/Tile: the textures repeats itself outside of its boundaries.
- Mirror: the texture repeats itself but mirrors the out of bound axis.
- Clamp/Clamp to Edge: the texture uses the bound value as a constant.
- Border/Clamp to Border: The texture uses a pre-defined border value.

### 6.1.3 Texture Values

This last main transformation is basically a lookup table from the texture space coordinates to an arbitrary value that we want to store in the image. Most commonly a RGB color but it could be anything: grayscale value, roughness, RGBA, 3D vector surface normal, etc.

However, there's also procedural texturing. It's not a lookup table but a different computation with the texture coordinates but this will be covered later on.

## 6.2 Image Texturing

The main issue to solve here is what happens when the surface to cover with a texture is much bigger (*magnification*) or much smaller (*minification*) than the texture size. This is a sampling and filtering problem and can be solved at a texture level (the input) or at a pixel level (the output).

If the input and output have a linear dependency (such as colors), then it doesn't matter where to do the filtering. However, for non-linear dependency cases (roughness, normals) filtering the input might cause *aliasing* so other techniques must be used.

### 6.2.1 Magnification

The idea is to use the values of the closest texels to the pixel in order to calculate the pixel texture value. Some common approaches are:

- **Nearest Neighbour**: just repeats the value of the closest texel. This results in a *pixelation* behavior.
- **Bilinear/Linear Interpolation**: the pixel takes the closest four neighbour texels within the two axes and interpolates in both dimensions. There is no pixeling but it's blurry.
- **Bicubic Interpolation**: more expensive but even less pixeling. Can be done in addition to some smoothing curves such as the *smoothstep* curve or the *quintic* curve.

### 6.2.2 Minification

When many texels contribute to a single pixel, it's very expensive (almost impossible) to know the exact contribution in real time. That's why we have some techniques to come up with a value.

- **Nearest Neighbour**: such as in magnification, we could take the value of the texel that's exactly in the center of the pixel. This causes a very ugly aliasing.
- **Bilinear Interpolation**: just as in magnification, we can apply the same concept but will produce aliasing in most cases if there are more than 4 texels per pixel.
- **Mipmapping**: the idea is to reduce the size of texels until one of the dimensions has a ratio of 1:1 with the pixels. We start with the original texture as *Level 0* and from there we divide the texture size by 4 (each dimension by 2) to get the next mip level or *subtexture*. The new level texel averages its neighbour texels from the former level. The mipmap levels form a **mipmap chain**. The user can choose what filtering function to use (box, Gaussian, etc) with different results. If the texture is sRGB, it must be converted to linear space in order to calculate mipmaps. There is also an interpolation between levels given the parameter **level of detail** and it's called **trilinear interpolation**. One downside of mipmapping comes when looking at a texture nearly edge-on and one dimension stretches while the other doesn't. This causes *overblurring* due to the asymmetry.
- **Summed-Area Table**: precomputes a table where each entry SAT[x,y] stores the sum of all texel values in the rectangle from (0,0) to (x,y). Any axis-aligned rectangular footprint can then be evaluated in constant time using four lookups and inclusion-exclusion: `sum = SAT[x2,y2] - SAT[x1-1,y2] - SAT[x2,y1-1] + SAT[x1-1,y1-1]`. The main drawbacks are that it requires higher-precision storage (values accumulate, so 8-bit per channel overflows), the footprint must be axis-aligned, and it has poor cache coherency. Anisotropic filtering is now generally preferred.
- **Unconstrained Anisotropic Filtering**: In this case the back-projection of the pixel's cell is used for two things. The shorter dimension sets the mipmap level and the longer dimension sets the anisotropy axis. The ration between the dimensions will dictate the amount of samples taken from the mipmaps.

## 6.5 Material Mapping

Instead of having constant values for properties like color, roughness or height, these material values can be bound to a texture map, creating a relationship between the object coordinate and the material properties values. Even more, the shader programs can use these values to choose a branch and make conditional computing for some specific materials. We must remember that the antialiasing and filtering is non-trivial for materials where the relationship between input and output is non-linear.

# 9. Physically Based Shading

## 9.1 Physics of Light

Thinking the light as a wave, physics describes it as the propagation of two fields that are orthogonal to each other and to the direction of propagation. These fields are the electric and the magnetic. The energy asocciated to the wave is proportional to the amplitud of each field, but as they're both related, we'll take the magnetic contribution in function of the electric field as well. Then, we know that the energy of the wave is proportional to the squared amplitud of the electric field. We take the electric field and not the magnetic as the former has much more incidence in the interaction between light and object illumination.

Although it might seem that the quadratic dependency of the energy is creating energy when we sum two waves in phase, in reality the energy is still conserved as in other place the interference is destructive.

*I didn't get this. If we have two light waves in the void that are in phase, then where is that "other place" that compensate the idea of the total energy being more than the sum of the parts?*

In the real world, the waves are almost perfectly random, so the interference is neither constructive nor destructive but linear dependent on the amount of waves.

A light wave contains many wave lengths all in one, but each one of them interacts in its own way when hitting some particle, scattering at some level depending on particle or material properties, mainly in the original axis of propagation.

### 9.1.2 Media

The media in which the light travels has an **index of refraction (IOR)** that plays a significant role when the light changes media. One property of the media is the **absorption**, the capacity to absorb light, attenuating it at an exponential rate (Beer-Lambert law). Both IOR and absorption vary with the wavelength.

Both scattering and absorption are scale-dependent (for example the water if we observe a glass or the sea).

### 9.1.3 Surface

A surface is a two dimensional interface separating two volumes with different IOR. When studying the incidence of the light in a surface, there are two main factors: **the substances in the volumes and the surface geometry**. 

Assuming a flat plane as the surface geometry, let's analyze the influence of the volume material. As a consequence of the electric field being continuous across the surface, the scattered wave preserves the incident direction. Then, the wave divides in two, the **transmitted/refracted wave** and the **reflected wave**. Also the frequency is conserved and the phase velocity changes proportionally to the volumes IOR ratio. As a result of all this, the reflected wave has the same angle with the normal as the incident one but the refracted wave has an angle that follows **Snell's Law**

**Why does the E-field boundary condition imply this?** The condition must hold at *every point on the surface* and at *every moment in time* simultaneously. This means the component of the wavevector parallel to the surface (`k_∥`) must be identical for the incident, reflected, and transmitted waves — otherwise there would be points/times where the condition fails. This is called **phase matching**.

- **Law of reflection**: the reflected wave is in the same medium so `|k|` is unchanged. Matching `k_∥` with the same magnitude forces `θ_reflected = θ_incident`.
- **Snell's Law**: the transmitted wave is in a different medium so `|k| = nω/c` differs. Matching `k_∥` across the two magnitudes gives `n₁ sin(θ_i) = n₂ sin(θ_t)`.

So the boundary condition doesn't preserve the incident direction — it enforces phase matching along the surface, and the reflection/refraction angles are the geometric consequence of that.

Regarding the surface geometry, we know that there is no perfectly planar surface. They all have irregularities and the size of these is what dictates the effect on light. Always comparing to the light wave length, if they are too big the surface has no local irregularities, it just tilts. If they are too small, they have no effect. Finally, if it's in range of 1-100 wavelengths, there is **diffraction** that we'll ignore.

When the irregularities are too small and we have to render them, we handle the scattering statistically and call this property **roughness**.

### 9.1.4 Subsurface Scattering

We can model the subsurface scattering locally or globally and this depends on the material properties and the **scale of observation**. When the entry-exit distances are small relative to the shading scale, they can be treated as zero and we can use a local shading model. This is handled by the **specular term** (reflection) and the **diffuse term** (subsurface scattering).
When entry-exit distances are large relative to the shading scale, some specialized global subsurface scattering technique is required.

## 9.3 BRDF

All in all, PBR's goal is to calculate the radiance of light at a given point and a given direction of propagation. Thus we have

$$L_i(c, -\mathbf{v})$$

with $L$ the **radiance**, $c$ the **camera position** and $-\mathbf{v}$ the **direction along the view ray**. We use $-\mathbf{v}$ because the direction argument in $L(x, \mathbf{d})$ describes where the light is *coming from* at $x$. Since $\mathbf{v}$ points from the surface toward the camera, $-\mathbf{v}$ points from the camera back toward the scene — the direction from which light arrives at $c$.

A *participating medium* affects radiance by absorption or scattering. Assuming for now that it's not present, the radiance emitted from the surface point is the same that arrives at the camera. Thus,

$$L_i(c, -\mathbf{v}) = L_o(p, \mathbf{v})$$

with $p$ the intersection of the view ray with the closest object surface.

It's our job now to calculate $L_o$. We could have emissive surfaces but most of the time the source of the radiance will be a reflection of a light source radiance. We'll leave transparency and global subsurface scattering aside, focusing only on **local reflectance** (i.e. reflection and local subsurface scattering). This local reflectance only depends on the light direction $\mathbf{l}$ and the outgoing view direction $\mathbf{v}$, and it's quantified by the **bidirectional reflectance distribution function** (BRDF). Worth mentioning that we'll actually cover *spatially varying* BRDFs (non-uniform surface).

Moreover, the two direction vectors span four degrees of freedom — two **elevation angles** with the normal and two **azimuth** angles with the tangent. For **isotropic** BRDFs, only the *relative* azimuth matters: the surface is rotationally symmetric around $\mathbf{n}$, meaning rotating both $\mathbf{l}$ and $\mathbf{v}$ by the same angle around $\mathbf{n}$ doesn't change the result. This reduces the DoF to 3.

One last assumption: we're not working with fluorescence nor phosphorescence, so the wavelength at incidence and outgoing is the same. As different wavelengths have different reflectances, we treat them as a *spectrally distributed value* — the way to go in real-time rendering.

Thus, we incorporate the BRDF to compute $L_o(p, \mathbf{v})$, omitting $p$ for simplicity:

$$L_o(\mathbf{v}) = \int_{\mathbf{l} \in \Omega} f(\mathbf{l}, \mathbf{v})\, L_i(\mathbf{l})\, (\mathbf{n} \cdot \mathbf{l})\, d\mathbf{l}$$

Worth noticing that when $\mathbf{l}$ is under the hemisphere we should not compute the equation, and when $\mathbf{v}$ or $\mathbf{n}$ are, the dot product must be clamped (with an epsilon or *soft clamped* to avoid artifacts).

Every BRDF has two constraints. The first, and more important in offline rendering, is **Helmholtz reciprocity**: $f(\mathbf{l}, \mathbf{v}) = f(\mathbf{v}, \mathbf{l})$. The second is **conservation of energy**: outgoing energy can **never** exceed incoming energy (omitting emissive materials). In RTR we'll always want to approximate this.

We can use the **directional-hemispherical reflectance** $R(\mathbf{l})$ to measure energy loss for a given incoming direction $\mathbf{l}$:

$$R(\mathbf{l}) = \int_{\mathbf{v} \in \Omega} f(\mathbf{l}, \mathbf{v})\, (\mathbf{n} \cdot \mathbf{v})\, d\mathbf{v}$$

Note the cosine factor is $\mathbf{n} \cdot \mathbf{v}$ (the *outgoing* angle) since we integrate over all outgoing directions $\mathbf{v}$.

Due to Helmholtz reciprocity this equals the **hemispherical-directional reflectance** $R(\mathbf{v})$, and *directional albedo* is used as a blanket term for both:

$$R(\mathbf{v}) = \int_{\mathbf{l} \in \Omega} f(\mathbf{l}, \mathbf{v})\, (\mathbf{n} \cdot \mathbf{l})\, d\mathbf{l}$$

**Why are R(l) and R(v) the same and why "directional albedo"?** By reciprocity ($f(\mathbf{l},\mathbf{v}) = f(\mathbf{v},\mathbf{l})$), swapping the integration variable in $R(\mathbf{l})$ gives exactly $R(\mathbf{v})$ — they're the same mathematical function of a direction. So the reflectance doesn't depend on which end (in or out) you compute it from; it's just a per-direction quantity. "Directional albedo" is the unified name for this. Both output values in $[0, 1]$: $0$ is total absorption, $1$ is total reflection.

The **Lambertian** is the simplest BRDF — $f$ is constant, meaning the surface scatters light equally in all directions. Substituting a constant $f$ into $R(\mathbf{l})$:

$$R(\mathbf{l}) = \pi \cdot f(\mathbf{l}, \mathbf{v})$$

and thus:

$$f(\mathbf{l}, \mathbf{v}) = \frac{\mathbf{c}_{diff}}{\pi}$$

with $\mathbf{c}_{diff}$ the diffuse color or albedo.

**Breaking down the Lambertian derivation:**
- $f(\mathbf{l}, \mathbf{v})$ is the BRDF — a ratio of scattered radiance to incoming irradiance. $L_i(\mathbf{l})$ is separate: the incoming radiance at the surface from direction $\mathbf{l}$ (i.e. the light sources). They combine in the rendering equation.
- Since $f$ is constant, pull it out of $R(\mathbf{l})$: $R(\mathbf{l}) = f \int_\Omega (\mathbf{n} \cdot \mathbf{v})\, d\mathbf{v} = f \cdot \pi$, because integrating $\cos\theta$ over the hemisphere gives $\pi$.
- We want $R(\mathbf{l}) = \mathbf{c}_{diff}$ (the surface reflects its albedo, no more). Setting $f \cdot \pi = \mathbf{c}_{diff}$ gives $f = \mathbf{c}_{diff}/\pi$.
- The book's "1/π factor is caused by integrating cosine over the hemisphere yielding π" means exactly this: $\pi$ appears as a multiplier after the integral, so we put $1/\pi$ in $f$ to cancel it and conserve energy. Your interpretation is correct.
