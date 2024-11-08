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

Four pictures are worth four thousand words:
<div style="display: flex; flex-wrap: wrap; justify-content: center;">
    <img src="https://github.com/user-attachments/assets/1d475466-a588-4413-b2af-ceaaf2594404" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-09">
    <img src="https://github.com/user-attachments/assets/6522acea-bbf1-45f9-b173-8d4ef2c2d389" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-11">
    <img src="https://github.com/user-attachments/assets/abad6bc1-028d-4575-8621-3d228bb0af0e" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-13">
    <img src="https://github.com/user-attachments/assets/d3fd25cd-9137-4212-b640-04047c4b84a6" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-18">
</div>

<img src="https://github.com/user-attachments/assets/14c31d86-a9ba-439c-a82f-5d87e19f13ee" width="50%" height="50%" alt="screenshot_2024-11-08_22-57-21">

```xml
<NearPlane>-1 1 -1 1 20</NearPlane> // broken in spheres_mirror.xml
```
