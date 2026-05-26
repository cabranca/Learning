# 2. The Graphics Rendering Pipeline

The graphics rendering pipeline (a.k.a. the pipeline) is a sequence of stages that run in parallel and are dependant of the previous stage. This means that the pipeline is as slow as its slowest stage.

There are 4 big stages in the graphics pipeline, each one divided in many smaller steps:

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

### Vertex Shading

The vertex shading historically used to compute color per vertex (Gouraud shading). Now the name stayed but in reality it performs transformations on the vertex positions to clip space (model → view → projection). Moreover, the Vertex Shader can output arbitrary data for the Pixel Shader to get as input and use in its computing (Normals, Tangents, Texture Coordinates). These outputs are interpolated across the primitive before reaching the Pixel Shader, which is what makes per-vertex data usable per-pixel.

# 3. The Graphics Processing Unit

## Data-Parallel Architectures

The GPU has **shader cores** that have to compute the same program, so the instructions will be the same. That's why the SIMD architecture fits perfectly. For the same instructions, we can process multiple data (e.g. fragment shading). 

Each program execution is called **thread** and they group into **warps/wavefronts**. This is useful because even though the arithmetic operations are fast enough, when the threads have to fetch data from memory, the whole core is stalled. Instead of waiting, the GPU switches context to another warp and starts over with the following threads. By the time all the warps are stalled, ideally the first warp will be ready to continue execution after fetching the data from memory. This technique is called **latency hiding**.

To ensure parallelism, I deduce that avoiding conditional execution with whiles and ifs is a **must**. The precise term for the problem is **thread divergence** (or branch divergence): when threads in the same warp take different branches, the warp executes **both paths serially**, masking inactive threads. So `if`/`while` aren't forbidden — they're only costly when threads in the *same warp* diverge. Uniform branches (all threads take the same path) are essentially free. **Rule of thumb**: branches on **uniforms** or **spatially coherent data** (e.g. screen position, since GPUs pack nearby pixels into the same warp) are cheap; branches on noisy per-fragment data (texture samples, random values) are expensive. Apart from that, there seems to be a sweet spot between instructions and *occupancy* (amount of warps). Also the amount threads are dependant on the registers they need. The more registers they need, the less threads can be created, resulting in fewer warps. Occupancy is also limited by **shared memory** usage per block, not just registers.

## GPU Pipeline Overview

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

## The Programmable Shader Stage

Nowadays, there is a unified shader design for most of the shader programs, i.e. they have the same *isntruction set architecture*. A processor implementing this model is called a **common-shader** in DirectX and the GPU would have a **Unified Shader Architecture**.

The shader languages usually have C-Style syntax and the most common are DirectX *High-Level Shading Language* (HLSL) and *OpenGL Shading Language* (GLSL). Even more, the HSLS shaders can be compiled to VM bytecode or *intermediate language*.
Shaders can be compiled offline of the main rendering program.
