---
layout: default
title: "The Pitfalls of Porting Code to the GPU"
---


## Article Contents
When writing code for a platform for the first time, it's easy to assume that the platform's standards won't change too much from what you are used to.
The logic usually stays the same, regardless of what programming language, compiler/interpreter, and OS you use.

This is where GPUs differ from this "standard", in that when you write a shader, it might not be executed the same on a different machine, or on the same machine with a different OS or different version of the same OS.


## Reason For The Topic
I chose this topic because, as a relative beginner to programming for GPUs, there are a few mistakes I've made that took me a long time to notice. Those mistakes caused problems way later down the line, in different places, making those mistakes a lot harder to find and fix.

These mistakes are not always obvious, and sometimes they're not even mistakes on some platforms. Nvidia GPUs have different standards for things like memory alignment, priorities, profiling etc, than AMD GPUs for example.


## The Problem
I decided to try making a 3D CSG (Constructive Solid Geometry) editor, using SDF (Signed Distance Fields) to define the shapes as mathematical functions. I would sample the value of the combined objects in a 3D grid of points, and then pass the data through a Marching Cubes algorithm to build the mesh, which would then be sent to a very simple shader to draw on screen.

Here's a simplified version of the C++ function that generates vertices and indices for their draw order using the Marching Cubes algorithm
(The original function is almost 200 lines, so I severely simplified for the sake of conciseness)
```cpp
MeshData build_mesh(VoxelGrid voxels) {
    vector<vec3f> vertices;
    vector<int> indices;
    for (const auto& [position, value] in voxels) {
        // Calculate the "zero point" between each corner of the voxel
        vec3f new_vertices[] = calculate_vertex_position_in_voxel(position, value);

        // This function is the meat of the algorithm.
        // Its main purpose is to search through a table with 256 possible vertex orders
        // The very large size of it is why I obfuscated it here.
        // You can easily find a "connection table" for Marching Cubes
        int new_indices[] = calculate_indices_for_voxel(position, voxels);

        // Some vertices might not be used if the "zero point" of the corners isn't between them
        vertices.emplace_back(eliminate_unused_vertices(new_vertices));
        indices.emplace_back(new_indices);
    }
    return MeshData{ vertices, indices };
}
```
This algorithm doesn't have any big problems, it isn't too slow, and it's easily multithreaded.

Problems do start appearing if we need to run this algorithm in real time, generating the mesh every frame before drawing it. The amount of data that we need to send to the GPU to be drawn increases exponentially with the resolution of the voxel grid.

Uploading data to the GPU is very slow, and if we want this algorithm to run in real time, this speed (or lack there-of) can pose a problem on any hardware, even on higher end machines, if that machine has a slow bus.


## The Solution
The most obvious solution to the problem of transfer speeds between the CPU and GPU, is to circumvent having to do this transfer entirely.
A bonus for this solution is that running the mesh generation code on the GPU means we get even better multithreading almost for free, because GPUs are specifically made to run a piece of code on a large amount of data all at once.

```glsl
```


## Unforeseen Problems
