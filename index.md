---
layout: default
title: "The Pitfalls of Porting Code to the GPU"
---


## Article Contents
When writing code for a platform for the first time, it's easy to assume that the platform's standards won't change too much from what you are used to.
The logic usually stays the same, regardless of what programming language, compiler/interpreter, and OS you use.

This is where GPUs differ from this "standard", in that when you write a shader, it might not be executed the same on a different machine, or on the same machine with a different OS or different version of the same OS.

This article will assume you have at least some basic knowledge of C++, Modern OpenGL, GLSL and matrix/vector math. If you know how to draw a cube and a flying camera using OpenGL, you should be fine. You can follow learnopengl.com for the first few chapters to get a good grasp of the basics.


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
A bonus for this solution is that running the mesh generation code on the GPU means we get all of the advantages of parallelism on the GPU, because GPUs are specifically designed for this kind of work.

This is a simplified version of the compute shader that generates the vertex positions.
```glsl
// Buffer binding for the vertex data
layout(std430, binding = 0) buffer VertexData {
	vec3 vertex_data[];
};


void main() {
    // Get the position
	uvec3 pos = gl_GlobalInvocationID.xyz;

	// Get the voxel data
	float current_voxel_data[8] = generate_voxel_values(pos);

	// Loop through all 8 corners of the voxel and check if they are inside or outside the surface
	// If they are inside, add them to the vertex list
	uint indices_order = 0;
	for (int i = 0; i < 8; i++)
		if (current_voxel_data[i] <= 0.0f)
			indices_order |= (1 << i);

	// Generate the zero point between each corner of the voxel
	vec3 verts[12];
	gen_voxel_verts(verts, pos, current_voxel_data);

	// Add the vertices to the vertex list
	uint indices_start = indices_order * 15;
	uint output_index = data_index(pos) * 15;
	for (int i = 0; i < 15; i += 3) {
		uint target_index = output_index + i;
		// If the index is -1, it means that the vertex is not used
		if (indices[indices_start + i] != -1) {
			// Add the face, made of 3 vertices, to the vertex list
			vertex_data[target_index] = verts[indices[indices_start + i + 2]];
			vertex_data[target_index + 1] = verts[indices[indices_start + i + 1]];
			vertex_data[target_index + 2] = verts[indices[indices_start + i]];
		}
	}
}
```

This way, the `vertex_data` array is already on the GPU, and when we want to draw the mesh, we just bind them to the appropriate locations and draw them.

Simply binding the buffers puts a lot less strain on the bus, and the CPU only needs to send the draw commands, which takes considerably less time than sending millions of vertices.

Creating the buffers is pretty straightforward:
```cpp
// First, we create the buffer and bind it to work on it
glCreateBuffers(1, &vertex_buffer);
glBindBuffer(GL_SHADER_STORAGE_BUFFER, vertex_buffer);

// We allocate the memory for the buffer, but we don't send any data to it, by providing nullptr as the argument for the data
glBufferData(GL_SHADER_STORAGE_BUFFER, sizeof(vec3) * num_vertices, nullptr, GL_STATIC_DRAW);
```

After creating the buffer, we don't need to change it again, unless the size of the mesh changes, so when we want to use it, we can just bind it to the appropriate location, end either call the compute shader to generate the data, or just draw it if the data is already there.
```cpp
// Bind the buffer to the appropriate location
glBindBuffer(GL_SHADER_STORAGE_BUFFER, vertex_buffer);
// Here it's assumed the location is 0, but it can be anything
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 0, vertex_buffer);

// Calculate how many work groups we need to dispatch
vec3 dispatch_size = voxel_grid_size / work_group_size;
// Dispatch the compute shader
glDispatchCompute(dispatch_size.x, dispatch_size.y, dispatch_size.z);

....

// Assuming the buffer is still bound in the same way, we can draw it like this
// NOTE: In OpenGL 4.2 and up, the buffer must be bound to a Vertex Array Object (VAO) before drawing
glDrawArrays(GL_TRIANGLES, 0, num_vertices);
```

This GPU implementation of the algorithm is a lot faster, and more elegant than the previous one, but it has a few problems of its own.


## Unforeseen Problems
Now that we have the algorithm implemented on the GPU, we can run our code and notice that there are a few things wrong. The mesh either isn't getting drawn, or there are triangles drawn from garbage data everywhere on the screen, and there are no errors in the console.

If you're lucky, you might see a shape that looks like the one you wanted to draw, popping through the rest of the garbage, but it's not quite right.


### Extra Triangles
First of, let's look at those triangles stretching in weird directions.

![extra-triangles](extra-triangles.png)

If you were paying attention earlier when writing the compute shader, you might have noticed that when we don't use a vertex, we don't just leave it out of the vertex list, we simply skip doing anything with it.

This means that there are a lot of vertices in the buffer that never get touched, and so, vertices that aren't generated by the compute shader, will have in them the garbage data they started with.

Thankfully this first problem has a relatively simple fix.
We need to clear the buffer before writing anything to it, so that we don't have any garbage data in it.
Because this clear shader is so simple, I'll show it here in its entirety:
```glsl
#version 450 core

// We don't need to worry about working in 3D space here, so we can just use a 1D work group
layout(local_size_x = 64, local_size_y = 1, local_size_z = 1) in;

// The buffer we want to clear
layout(std430, binding = 0) buffer buff {
    uvec3 data[];
};

void main() {
	// Set the value of each element in the buffer to 0
    data[gl_GlobalInvocationID.x] = uvec3(0);
}
```
Now all we need to do is bind the vertex buffer to the appropriate location, and dispatch the clear shader.
```cpp
// Bind the buffer to the appropriate location
glBindBuffer(GL_SHADER_STORAGE_BUFFER, vertex_buffer);
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 0, vertex_buffer);

// Dispatch the clear shader
glDispatchCompute(num_vertices / 64, 1, 1);
```
You might notice that the buffer is of type uvec3, meaning it contains 3 unsigned integers instead of 3 floats like the vertex buffer we're clearing.
This doesn't actually matter, because floats and unsigned integers are both 32 bits long, and adding the unneeded complexity of floats doesn't help anything, so setting an unsigned integer to 0 is the same as setting a float to 0.0f.

Good, we fixed, and when we run it, we get a mesh that looks like this:

![still-broken](still-broken.png)

Wait, it's still broken? What's going on?


### Memory Alignment
At least we don't get all those garbage triangles anymore, but now there's just a piece of the mesh missing.
But you should notice that the missing piece is always in the same place, and it's always the same size, at least if the size of the voxel grid is the same.

Well, if you were to follow instructions you found online for how to use compute shaders, you might want define buffers in the compute shader like this:
```glsl
layout(std430, binding = 0) buffer VertexData {
	vec3 vertex_data[];
};
```

This is technically not wrong, and if you have an NVidia GPU, or one of a few AMD GPUs as well, you might even think this works, but it doesn't.
This is because of the way NVidia GPUs handle memory alignment, and because OpenGL doesn't specify a concrete standard for how memory should be aligned, which leaves some room for interpretation by the GPU manufacturers.

The biggest problem comes from std430, which is a layout qualifier that technically only specifies that the memory should be tightly packed, but the OpenGL standard allows for different rules specifically for vec3, which is what we're using for the vertex buffer.

Intel will still align a buffer of vec3 to 16 bytes, which means it will look something like this:
```
[ X Y Z _ X Y Z _ X Y Z _ X Y Z _ ]
```
The `_` represents an empty space the same size as a float, which is 4 bytes.

A much more concrete standard is std140, which specifies that vec3, and many other types as well, should be aligned to 16 bytes, regardless of their size.
This means the alignment will still be the one of vec4, so there will still be that empty space, but at least it will be consistent across all GPUs.

Now, all we need to do to fix this alignment issue is to allocate memory for a buffer of vec4 instead of vec3.
Here's the line that we need to change, side by side with the old one:
```cpp
// Old
glBufferData(GL_SHADER_STORAGE_BUFFER, sizeof(vec3) * num_vertices, nullptr, GL_STATIC_DRAW);

// New, with the size of vec4 instead of vec3
glBufferData(GL_SHADER_STORAGE_BUFFER, sizeof(vec4) * num_vertices, nullptr, GL_STATIC_DRAW);
```

After this simple change, we get a mesh that looks like this:

![after-memory-alignment](after-memory-alignment.png)
