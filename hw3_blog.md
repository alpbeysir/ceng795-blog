# CENG795 Advanced Ray Tracing - Homework 3

## Failed

I could not complete this homework on time. I will explain the parts I completed. I used one late day.

## Multisampling

Multisampling is an integral part of this homework. The effects are all based on multisampling and don't look good without it. Fortunately, it is surprisingly simple to code. I formed a virtual square for each of the pixels in the image. I divided this square into subsquares based on the scene file parameters. Then, using a random number generator I created floating-point sampling position in each subdivision of the square:

```cpp
auto color = glm::vec4(0.0f); // initialize color as black

for (int k = 0; k < sample_length; k++) { // iterate each row of the subsquare
    for (int l = 0; l < sample_length; l++) { // iterate each element of the row
        // create sampling positions
        const float sampling_x = (static_cast<float>(k) + random_uniform_float() + 0.5f) * sample_grid_length;
        const float sampling_y = (static_cast<float>(l) + random_uniform_float() + 0.5f) * sample_grid_length;

        // this part is unchanged
        glm::vec3 pixel_position = plane_pixel_position(current_camera, i, j, sampling_x, sampling_y);
        const Ray ray = Ray::from_to(current_camera.position, pixel_position, random_uniform_float());
        color += trace(scene, ray, scene.max_recursion_depth + 1, glm::vec4(1.0f), false,
                                 glm::vec4(0.0f));
    }
}
```

After this aggregate color value is obtained, I divide it by the number of samples given in the scene file and set that value as the pixel color:

```cpp
color /= current_camera.num_samples;
image[i * current_camera.image_height + j] = color;
```

The code above averages the obtained color value from the ray tracer, allowing us to use distribution/randomized effects.

## Depth of Field

Failed :( But I was able to do it after the deadline!

## Area Light

Although there are some issues with my implementation, the results mostly match the reference images. I have yet to discover my mistakes.

<img src="https://github.com/user-attachments/assets/1bd46bc4-87f0-43c8-a8fc-403c27f10c94" width="50%" height="50%">

Area light code is basically the same as point light code, except for one difference. In area lights, before doing the computation with the light position a random position in the area light's extent is taken instead of the light's actual position. 

The code below is run for each ray that hits an object:

```cpp
for (const auto &area_light: scene.area_lights) { // iterate every area light in the scene
    const auto [u, v] = get_onb(area_light.normal); // construct orthonormal basis of area light
    const auto p = area_light.position + (u * random_uniform_float() + v * random_uniform_float());

    // after we get the light position, the shading is exactly the same as point lights
    Ray shadow_ray = Ray::from_to(hit_point + norm * scene.ray_epsilon, p, incoming_ray.t);
    if (is_shadowed(shadow_ray, scene))
        continue;

    const auto l = shadow_ray.direction;
    const auto r2 = dot(l, l);
    const auto I = area_light.radiance * (area_light.size * area_light.size) / r2;

    color += do_diffuse_shading(l, I, norm, material);
    color += do_specular_shading(l, I, norm, material, incoming_ray);
}
```

## Motion Blur

Motion blur is implemented by adding a t parameter to the Ray struct:

```cpp
struct Ray
{
    glm::vec3 start;
    glm::vec3 direction;
    float t; // simulates the 'time' this ray came into the camera shutter

    void apply_motion_blur(const glm::vec3& motion_vector) { // this function is used to transform a ray
        start -= motion_vector * t;
    }
}
```

This parameter is set as a random float between 0 and 1 for each ray leaving the camera. When doing intersection checks against meshes and spheres, the parameter is used to transform the ray. This transformation of the ray simulates the movement of the object in the scene. 

```cpp
for (const auto &mesh_info : scene.mesh_infos)
{
    Ray local_ray = Ray::transform(ray, mesh_info.inverse_transformation);
    local_ray.apply_motion_blur(mesh_info.motion_blur); // use the function defined above to 'move' the ray

    // do the usual collision stuff
    if (const auto [t, tri] = bvh_get_collision(scene, mesh_info.bvh, local_ray); t_min > t)
    {
        // ...    
    }
}
```

Additionally, if a ray bounces the parameter is kept the same. This ensures consistency of reflections and actual objects. 

## Roughness

The roughness parameter of a material modifies the look of its reflections. Basically, a reflected ray is nudged in a bit of a different direction to give the effect of imperfections on the surface. The reflection direction is chosen randomly for each bounce so we need multisampling for this effect to look good.

I implemented this by adding a member function to my Ray struct definition:

```cpp
void apply_roughness(Ray &ray, const float roughness) { // this is called for each reflected ray
    if (roughness <= glm::epsilon<float>()) return; // check if roughnes value is too small to make an effect (roughness is set to zero when not given)

    const auto [u, v] = get_onb(ray.direction); // computer orthonormal basis for ray direction
    ray.direction = normalize(ray.direction + roughness * (random_uniform_float() * u + random_uniform_float() * v)); // nudge the ray a bit
}
```

<img src="https://github.com/user-attachments/assets/4045db0a-5602-4efb-a180-54cc6f01974b" width="50%" height="50%">


