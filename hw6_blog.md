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


