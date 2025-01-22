# CENG795 Advanced Ray Tracing - Homework 6

## BRDF

The BRDF's in this homework bring more realistic models of light bounces to my ray tracer. The BRDF part was pretty straightforward.
A BRDF takes in some parameters about the material and the hit information returns how the object should reflect the light.
The function definition is based on this:

```cpp
glm::vec3 do_brdf(const Material &material,
                  const glm::vec3& textured_diffuse,
                  const glm::vec3& textured_specular, // the first three parameters are about the object itself
                  const glm::vec3& wi,
                  const glm::vec3& norm,
                  const glm::vec3 &wo) { // these three are about the hit situation
    const auto& brdf = material.brdf;
    const auto diffuse = textured_diffuse * std::max(epsilon, dot(norm, wi)); // the diffuse is the same regardless of the brdf type
    switch (brdf.type) {
        case OriginalBlinnPhong: {
        }
        case OriginalPhong: {
        }
        case ModifiedBlinnPhong: {
        }
        case ModifiedPhong: {
        }
        case TorranceSparrow: {
        }
        default:
            throw std::runtime_error("unknown brdf type");
    }
    return diffuse + specular;
}
```

Now that I had the function calculating the BRDF, I integrated it to the existing shading routines. At first there was no path tracing so the only changed part was the direct lighting.
In my direct lighting code, I have for loops iterating each light type in the scene. For each light the calculations may be different but in the end each calculates the same two pieces of information:

- Incoming light direction
- Intensity

This similarity makes it easy to swap out the shading functions.

Before I had separate functions for diffuse & specular, now I just call `do_brdf()`:

```cpp
  for (const auto &point_light: scene.point_lights) {
      // ... do calculations
      const glm::vec3 l = shadow_ray.direction;
      const glm::vec3 I = point_light.intensity / r2;

      // Before
      color += do_diffuse_shading(material, l, I, norm);
      color += do_specular_shading(material, l, I, norm, incoming_ray);
      // After
      color += do_brdf(material, diffuse, specular, normalize(l), norm, -normalize(incoming_ray.direction)) * I;
  }
```

Unlike before, as a small optimization I did not pass `I` as a parameter because I can just multiply the result with it later.

### Results

Modified Blinn-Phong:

![brdf_blinnphong_modified](https://github.com/user-attachments/assets/6f9df9c0-61d8-471c-a77d-09634d67bfd9)

Normalized Modified Blinn-Phong:

![brdf_blinnphong_modified_normalized](https://github.com/user-attachments/assets/21ac92fb-6bcf-4ba3-bcf5-005c8fdea79c)

### Honorable Mentions

While implementing BRDF support I accidentally ignored all texture data. This led to disappointing results in my path traced render of veach-ajar:

![VeachAjar_path](https://github.com/user-attachments/assets/1f8321a7-a0e6-4c2c-bb08-8187a1427913)

## Path Tracing & Object Lights

These two sections I handled together because path tracing does not work without object lights.

At first the concept of path tracing seemed daunting and hard to approach. After all, it allows for cool and realistic effects that are hard to reproduce otherwise.
I think that a good way to think about it is sampling an environment light, but rather than a projected texture color **the scene is used as a color value.**

```cpp
glm::vec4 trace(const Scene &scene,
                const Camera& camera,
                const Ray& ray, int depth,
                glm::vec3 energy, bool is_inside,
                const glm::vec3& absorption_coefficient) {

  // ... geometry handling

  // extra handling here
  if (camera.renderer_type == PathTracer) {
        const auto [dir, probability] = cosine_importance_sample(norm); // better than uniform sampling
        Ray global_ray(...);
        const auto global_illumination_result = trace(...); // this is the important part
  
        auto diffuse = material.diffuse;
        auto specular = material.specular;
        auto ambient = material.ambient;

        if (!out_hit_info.texture_ids->empty()) {
            apply_textures(...); // don't forget to apply the texture data
        }

        const auto brdf = do_brdf(...);
        color += glm::vec4(global_illumination_color * brdf, 0.0f);
    }
  
  // ... material & shading handling
  
  return color;
}
```

diffuse_test:

![diffuse_test](https://github.com/user-attachments/assets/d7aa1d2d-94a3-4c8f-a063-8ab4d46d2686)

### Object Lights

In standard path tracing, sampling object lights is literally one line of code:

```cpp
glm::vec4 trace(const Scene &scene,
                const Camera& camera,
                const Ray& ray, int depth,
                glm::vec3 energy, bool is_inside,
                const glm::vec3& absorption_coefficient) {

  // ... geometry handling

  // check if hit object is a light
  if (bool hit_light = length2(out_hit_info.radiance) > epsilon) {
      color += glm::vec4(out_hit_info.radiance, 0.0f);
      color.w += 1.0f; // used for 0/1 heuristic
  }

  // ... material & shading handling
  
  return color;
}
```

One caveat is that this will create a biased output when next event estimation is used. I dealt with that in the next section.

### Next Event Estimation

In effect, this was already implemented as direct lighting in our ray tracers. I just needed to disable it in case path tracing with no next event estimation was set for the camera.

```cpp
// trace.cpp, main routine

bool should_skip_direct_light = camera.renderer_type == PathTracer && !camera.next_event_estimation;
if (!is_inside && !should_skip_direct_light) { // this is added to disable it
    const auto shading_result = do_shading(scene, out_hit_info, ray) * energy;
    color += glm::vec4(shading_result, 0.0f);
}
```

One problem here is if the path tracing routine hits a light and its radiance is added to the color, the `do_shading` function will try to sample the same light again. This will cause bias.
I devised a weird way to fix this. I did not want to return an extra value from my trace function and use if statements to keep track of light hits.
Instead of returning `glm::vec3` from the trace function like normal, I changed it to `glm::vec4`. The `w` parameter tracks the number of lights hit. 

```cpp
// in trace.cpp, path tracing
const auto global_illumination_result = trace(...)
// if the 'number of lights sampled' is greater than zero and we have direct lighting
bool should_discard = global_illumination_result.w > epsilon && camera.next_event_estimation; 
if (!should_discard) { // may discard the sample
    // .. apply path tracing to color
}
```

#### Caustics

The avid reader may ask how this deals with caustics. I just reset the 'light counter' if we refract from a dielectric:

```cpp
// trace.cpp, material handling

if (material.type == MIRROR) {
    // .. mirror
}
else if (material.type == DIELECTRIC) {
    if (inside_sqrt > 0) {
        // .. math

        auto reflection_color = trace(...);
        color += reflection_color;

        auto refraction_color = trace(...);
        color += refraction_color;

        // caustics
        color.w = 0.0f;
    } else {
        // total internal reflection

        // .. math
                
        auto reflection_color = trace(...);
        color += reflection_color;
    }
}
```

glass_importance_nee_weighted_revised:

![glass_importance_nee_weighted_revised](https://github.com/user-attachments/assets/b37ff1d8-3c4e-4060-80c3-cf4dad1efe56)

#### Object Lights (NEE)

*I think this is the weakest part of my implementation. It does not exactly match the ground truth outputs.*

To make the last step of my ray tracing journey more exciting, I had the brilliant idea of conjuring up two new methods of sampling object lights:

**Sphere:**

```cpp
const auto local_point_on_sphere = sphere.position + glm::sphericalRand(sphere.radius + scene.ray_epsilon); // pick random point on sphere's surface
const auto transformed = sphere.transformation * glm::vec4(local_point_on_sphere, 1.0f); // transform it to global space
Ray shadow_ray = Ray::from_to(hit_point + norm * scene.ray_epsilon, glm::vec3(transformed), incoming_ray.t); // send ray towards that point
```

This seems to work well enough and is more performant than the original method. It probably introduces bias of some sort.

cornellbox_jaroslav_glossy_area_sphere:

![cornellbox_jaroslav_glossy_area_sphere](https://github.com/user-attachments/assets/bfe8635a-463c-43eb-8f61-c2494061a371)

**Mesh:** 

Since the sphere method works, I tried to adapt it to meshes with questionable results:

```cpp
const auto [bb_min, bb_max] = get_mesh_aabb(mesh_info);
const int face = static_cast<int>(random_uniform_float01() * 6); // pick random face
const float u = random_uniform_float01();
const float v = random_uniform_float01(); / pick random quad face coordinates

glm::vec3 point = bb_min;

switch (face) {
    // pick point on the selected random face
}

const auto local_point_on_box = point;
const auto transformed = mesh_info.transformation * glm::vec4(local_point_on_box, 1.0f); // transform to global space
Ray shadow_ray = Ray::from_to(hit_point + norm * scene.ray_epsilon, glm::vec3(transformed), incoming_ray.t); // send ray
```

I believe that this is neither uniform nor correct. A better method may be 'projecting' the AABB of the mesh to the point's 'view'.

cornellbox_jaroslav_glossy_area_small:

![cornellbox_jaroslav_glossy_area_small](https://github.com/user-attachments/assets/0481159b-d6d8-4469-98f2-97da2decd157)

### Importance Sampling

This code is pretty much exactly as described in the slides but I wanted to touch on a **crucial** mistake that I keep making.

- Multiplying by the probability instead of dividing.

I lost hours thinking I had gotten path tracing wrong when all along it was just this.

## Gallery

![VeachAjar_path](https://github.com/user-attachments/assets/69983abc-b733-4fff-9d0d-8af9d9421239)

![scienceTree_glass](https://github.com/user-attachments/assets/9cc7edaf-e653-4d82-948a-51476181b709)







