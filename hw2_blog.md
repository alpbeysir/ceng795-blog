# CENG795 Advanced Ray Tracing - Homework 2

This homework's topic was building on top of our existing ray tracer to add support for mesh instances and acceleration structures. 

## Acceleration Structures

The algorithm used here is named BVH which is used to divide a set of triangles in a certain way such that the number of triangle intersection checks for most rays is minimized.
This improves performance by orders of magnitude as in ray tracers, the main bottleneck is the number of triangle intersection checks.

### BVH Tree Generation

BVH at its core is a tree structure pre-computed before rendering. The node structure is defined below:

```cpp
struct BVHNode
{
    glm::vec3 bbmin, bbmax; // bounding box coordinates
    BVHNode *left, *right; // the children nodes
    int tri_start, tri_count; // the triangles "owned" by this node (indexes into the scene triangles list)
};
```
A node represents a cube-shaped portion of space containing some triangles in a mesh. This cube is split recursively along one of the main axes until only one or two triangles remain in it.

This is done by the function below which is run for each mesh at import-time (some details omitted for brevity):

```cpp
BVHNode *setup_node(Scene &scene, int tri_start, int tri_count)
{
	BVHNode *node = new BVHNode();
	node->tri_start = tri_start;
	node->tri_count = tri_count;

	set_node_bounds(scene, *node); // calculate node's AABB from its owned triangles

	auto left_count = split_node(scene, node); // find split point and reorder scene triangle array

	if (left_count == 0 || left_count == node->tri_count) // check if split failed, return node without splitting
		return node;

	node->left = setup_node(scene, tri_start, left_count);
	node->right = setup_node(scene, i, node->tri_count - left_count);

	if (node->left != nullptr && node->right != nullptr)
		node->tri_count = 0;

	return node;
}
```

Five pictures are worth five thousand words:

<img src="https://github.com/user-attachments/assets/1d475466-a588-4413-b2af-ceaaf2594404" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-09">
Level 0
<img src="https://github.com/user-attachments/assets/6522acea-bbf1-45f9-b173-8d4ef2c2d389" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-11">
Level 1
<img src="https://github.com/user-attachments/assets/abad6bc1-028d-4575-8621-3d228bb0af0e" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-13">
Level 2
<img src="https://github.com/user-attachments/assets/d3fd25cd-9137-4212-b640-04047c4b84a6" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-18">
Level 8
<img src="https://github.com/user-attachments/assets/14c31d86-a9ba-439c-a82f-5d87e19f13ee" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-21">
Level 16

This is for the input bunny.png. I made this using [BVHVisualization](https://github.com/MircoWerner/BVHVisualization/), an OpenGL and imgui program that allows for easy viewing of generated BVHs.
The images show the gradual segmentation of the bunny by the algorithm. At the early levels one can see the cube being split into two subcubes.

### BVH Intersection Check

So after we have this tree, how can we use it to make the ray tracer faster? The structure is used instead of iterating all triangles in a scene:

```cpp
// Before
for (const auto& tri : scene.triangles)
{
	float t = triangle_get_collision(scene.vertex_data, tri.indices, ray);
	if (t_min > t)
	{
		t_min = t;
		obj.type = COLLISION_OBJECT_TRI;
		obj.data.tri = &tri;
	}
}
```

```cpp
// After
const auto [t, tri] = bvh_get_collision(scene, scene.bvh, ray);
if (t_min > t)
{
	t_min = t;
	obj.type = COLLISION_OBJECT_TRI;
	obj.data.tri = tri;
}

// The new function iterating the BVH structure
std::pair<float, const Triangle *> bvh_get_collision(const Scene &scene, const BVHNode *node, const Ray &ray) {
	const Triangle *ret = nullptr;

	// This is the speedy part, if the ray does not intersect the bounding box
	// we can skip checking potentially millions of triangles at once
	if (node == nullptr || !aabb_get_collision(ray, node))
		return std::pair<float, Triangle *>(INFINITY, NULL); 

	// Otherwise we check if we can go into deeper nodes
	if (node->tri_count > 0) {
		// We can't, iterate through all triangles in this node (in the cube)
		float tmin = INFINITY;
		for (int i = node->tri_start; i < node->tri_start + node->tri_count; i++) {
			const Triangle &tri = scene.triangles[i];
			float cur_result = triangle_get_collision(scene.vertex_data, tri, ray);
			if (cur_result < tmin) {
				tmin = cur_result;
				ret = &scene.triangles[i];
			}
		}
		return std::pair<float, const Triangle *>(tmin, ret);
	}
	else {
		// We can, do it
		auto left_res = bvh_get_collision(scene, node->left, ray);
		auto right_res = bvh_get_collision(scene, node->right, ray);
		if (left_res.first < right_res.first)
			return left_res;
		else
			return right_res;
	}
}
```

This change shows performance improvements so big it does not even make sense to measure. Scenes over 10000 triangles become accessible for my ray tracer.

## Multithreading

Normally this may present a challenge but OpenMP comes to my rescue. This is my main loop sending rays for a camera:

```cpp
	#pragma omp parallel for // This OpenMP compiler directive here magically adds multithreading!
	for (int i = 0; i < current_camera.image_width; i++)
	{
		for (int j = 0; j < current_camera.image_height; j++)
		{
			Ray ray = Ray::from_to(current_camera.position, plane_pixel_position(current_camera, i, j));
			ray.direction = glm::normalize(ray.direction);
			auto color = trace(scene, ray, scene.max_recursion_depth + 1, glm::vec4(1.0f), false, glm::vec4(0.0f));
			image[i * current_camera.image_height + j] = color;
		}
	}
```

This achieved a moderate speedup for big scenes.

## Instanced Rendering & Transformations

### Loading & Processing

Implementing this was not an algorithmic challenge, more a software enginnering one. A huge part of the code had to be refactored to accomodate this feature.
Instancing is basically the input file saying: "I want another copy of this mesh, but somewhere else".
I wanted to keep much of the complexity of instancing at the parser and make minimal modifications to the ray tracer itself. Thus my architecture is as follows:

1. Start reading meshes and mesh instances from XML.
2. Process and 'squash' all transformations into one glm::mat4 for each mesh.
3. Create SquashedMeshInfo's from the flattened data.
4. Run using an array of these 'mesh infos'

SquashedMeshInfo represents all the information needed to render a mesh:
```cpp
struct SquashedMeshInfo {
    glm::mat4 transformation; // composite transformation
    int material_id;
    BVHNode* bvh; // the root node of this mesh's BVH
};
```

And this is the function that processes the original mesh data to create them:
```cpp
SquashedMeshInfo squash_mesh_info(int mesh_index, const Scene& scene) {
    auto mesh = scene.meshes[mesh_index];
    if (mesh.is_instance) {
        auto child = squash_mesh_info(mesh.base_id, scene); // recurse to get details of base mesh
        int material_id = mesh.material_id == -1 ? child.material_id : mesh.material_id; // material_id may be missing
        glm::mat4 transformation;
        if (mesh.reset_transform) { // take reset_transform into account
            transformation = mesh.transformation;
        }
        else {
            transformation = mesh.transformation * child.transformation;
        }
        return {transformation, material_id, child.bvh};
    }
    else {
        return {mesh.transformation, mesh.material_id, mesh.bvh};
    }
}
```

In the eyes of the ray tracing part, all meshes and instances are now the same thing.
I believe this architecture will make it much simpler in the future to add more data to meshes & mesh instances. 

### Rendering Transformed Meshes

Transforming all triangles of the mesh and then rebuilding the BVH is a huge waste of time. Instead, the cast ray is transformed into the local space of the mesh and BVH collision checks are performed there.

```
// Collision check
glm::mat4 inverse_transformation = glm::inverse(mesh_info.transformation);
Ray local_ray = Ray::transform(ray, inverse_transformation);
const auto [t, tri] = bvh_get_collision(scene, mesh_info.bvh, local_ray); // use local_ray here
```

The t parameter returned can be used normally to get the hit point of the ray.
However, there is a problem. The function bvh_get_collision returns the triangle hit but the normal is in local space. Thus, we need to transform the normal to world space:

```cpp
glm::mat4 inv = glm::inverse(mesh_info.transformation);
glm::mat4 inv_transpose = glm::transpose(inv);
norm = glm::mat3(inv_transpose) * local_norm;
```

The calculated value norm can finally be used for shading.

# Gallery

<img src="https://github.com/user-attachments/assets/06a5954f-45ac-4e0f-9c03-17f35a162008" width="50%" height="50%">
<img src="https://github.com/user-attachments/assets/ae404fa0-0fe5-489d-bdec-b9681238b848" width="50%" height="50%">
<img src="https://github.com/user-attachments/assets/70f8357f-660a-400d-b327-2d0b5c8705cf" width="50%" height="50%">
<img src="https://github.com/user-attachments/assets/4a890d73-d1f9-4a03-a9d8-523eefae2fe7" width="50%" height="50%">
<img src="https://github.com/user-attachments/assets/e4bbfa9c-233d-4770-a987-9dadfcf65586" width="50%" height="50%">

# Some Issues

I noticed two issues with my ray tracer which I hope to remedy by the next homework.

1. High poly meshes have small black 'dead pixels'.
2. The camera is distorted in some inputs. Weird.
