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

### Vertex Shading

The vertex shading historically used to compute color con vertex. Now the name stayed but in reality it performs transformations on the vertex positions given a model, a view and a projection. Moreover, the Vertex Shader can output arbitrary data for the Pixel Shader to get as input and use in its computing (Normals, Tangents, Texture Coordinates). 
