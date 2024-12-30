# CENG795 Advanced Ray Tracing - Homework 5

We are nearing the end of our ray tracing journey. This homework is about new light types and tonemapping.

## Directional Light

The directional light simulates sunlight, a light infinitely far away with no attenuation by distance.

First, I needed to modify how the code checked if the point should be shadowed.

Before:

```cpp
bool is_shadowed(const Ray &ray, const Scene &scene);
```

After:

```cpp
bool is_shadowed(const Ray &ray, const Scene &scene, const float min_t, const float max_t);
```

Before I would send a shadow ray to this function with a non-normalized direction equal to the difference between the hit point and the light. 
Afterwards, it would check if `t` was between `0 - 1`. This worked perfectly for the light types we had up until now.
But for directional lights, this function would need to check between `0 - ∞`. So I added two parameters for that.

The rest was trivial, sending a shadow ray in the opposite direction of the light.

```cpp
const glm::vec3 I = directional_light.radiance; // no attenuation by distance
```

![cube_directional](https://github.com/user-attachments/assets/76f3d12e-de82-4eae-89ec-f06e32eada2f)



