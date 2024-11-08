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


```xml
<NearPlane>-1 1 -1 1 20</NearPlane> // broken in spheres_mirror.xml
```
