# 2. The Graphics Rendering Pipeline

The graphics rendering pipeline (a.k.a. the pipeline) is a sequence of stages that run in parallel and are dependant of the previous stage. This means that the pipeline is as slow as its slowest stage.

There are 3 big stages in the graphics pipeline, each one divided in many smaller steps:

- Application
- Geometry Processing
- Rasterization
- Pixel Processing

## Application Stage

The Application stage is the CPU work. It can do previous calculations like collision detection or input handling. It prepares the geometry data for the GPU.

## Geometry Processing Stage

The Geometry Processing stageis divided into the following stages

- Vertex Shading
- Projection
- Clipping
- Screen Mapping

### Vertex Shading and Projection

The vertex shading historically used to compute color con vertex. Now the name stayed but in reality it performs transformations on the vertex positions given a model, a view and a projection. Moreover, the Vertex Shader can output arbitrary data for the Pixel Shader to get as input and use in its computing (Normals, Tangents, Texture Coordinates). 

### Optional Vertex processing

Depending on the HW, the output of the Vertex Shading could be processed by one or more of the following operations:

#### Tesellation

The tesellation generates vertices from the original set, the amount depending for example in how close is the object to the camera. The idea is to have a smoother surface (more vertices) when the object is close and a rougher surface (less vertices) when the object is far away. There are 3 sub-divisions of this process: **Hull Shader**, **Tesellator** and **Domain Shader**.

#### Geometry Shader

Older that tesellation, it's more common across GPUs. It consist in a simpler vertex generation, for example for particle generation. It can generate a quad from a point, making it easier to shade.

#### Stream Output

This is a more general step, the vertex output is taken as an array that can be processed by both CPU or GPU, tipically used for particle simulations.

### Clipping

Before passing the data to the Rasterization stage, we must be sure to only render what's visible, i.e. what's partially or totally inside the *view volume*. That's why it's so important to transform the vertex coordinates into the homogeneous space, so everything it's compared to the unit cube limits.

If a primitive is totally inside the unit cube, it's rasterized as it is. If it's totally outside, it's ignored. If it's partially inside, the outer vertex must be replaced with new vertices in the intersection of the primitive and the unit cube.

The user may also define extra clipping spaces. this is called **sectioning**. Finally the clipped vertices are transformed with a **perspective division** so they fit a *normalized device coordinates*.

### Screen Mapping

After clipping, the vertex coordinates are transformed to **screen coordinates**. If *x* and *y* joined with the *z*-coordinate, they make the **window coordinates**.

The mapping is done by a translation and a scale. The *x* and *y* coordinates fits the screen and the *z* component is mapped to an per-API arbitrary range. When all the transformations are done, these coordinates are passed on to the rasterizer stage.

As a side comment, the screen coordinate are floating point values even though we have an integer number for the amount of pixels per row and column. This means that the center of the pixel is calculated as its index plus 0.5.


## Rasterization

Also known as *scan conversion*, rasterization is the process of finding all the pixels within a primitive. It's the conversion from window coordinates (screen + z) to pixels on the screen. The rasterization has two steps: **triangle setup/primitive assembly** and **triangle traversal**.

The interesting part of rasterization is that one must define what it means for the pixel to be "inside" the primitive. Some of the heuristics are:

- Single sampling: if the center of the pixel is inside the primitive, then the whole pixel is.
- Supersampling/Multisampling antialiasing: same base idea but with more samples per pixel.
- Conservative rasterization: a pixel is inside the primitive if some part (no matter how much) of the pixel overlaps.

### Triangle Setup

Fixed-function HW is used here to perforom differential/edge equations. This can be used later in triangle traversal or mabye for interpolation of the shading data produced by the geometry processing stage.

### Triangle Traversal

This is the moment where the sampling is performed and the **fragments** are created, interpolating shading data from the original vertices. Tehre's also a *perspective-correct interpolation* (more on that later).

## Pixel Processing

### Pixel Shading

The Pixel Shading consists of processing the interpolated shading data to output one or more colors per fragment for the next step. While the rasterization is done by dedicated hardwired silicon, the pixel shading is executed in the GPU cores and it's programmable (what we call a pixel/fragment shader program). Many techniques are performed here such as **texturing** and **illuminations computing**.

### Merging

The merging step receives a color buffer as the input and must perform operations with this buffer and the current color buffer to see what's the next result for each pixel. Althought this step is not programmable, it's highly configurable and it's called ROP (Raster oprations pipeline/Render output unit).

One of the algorithms performed here is the **Depth/*z*-buffer comparisson**. The GPU compares primitives *z* value and keeps the lesser one because it means it's closer to the viewer. The good thing is that this is independent of the order, but it doesn't work out of the box for partially transparent primitives. They must be rendered in order after the opaques or just use another algorithm.

The **alpha channel** is asocciated with the color buffer and can be used in *alpha-testing* to discard or even merge pixel colors. This is a way to ensure that transparent primitives do not affect the *z*-buffer.

A **stencil buffer** can be used to filter the rendering, creating sections of rendering or no-rendering.

All in all, the **Framebuffer** contains all this buffers and all the operations at the end of the pipeline are called *raster operations* or *blend operations*. When the pipeline finishes its last stage, the frame buffer is presented as the *front buffer* on screen. To prevent the viewer from seeing how the pixels are re-drawn every frame, a *back buffer* is always being written while the front when is being read. This one of many techniques is called **double buffering**.

