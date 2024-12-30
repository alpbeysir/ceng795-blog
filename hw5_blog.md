CENG795 Advanced Ray Tracing - Homework 5

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

## Spot Light

The spot light is mostly the same as the point light:

```cpp
glm::vec3 I;
if (angle < spot_light.half_falloff_angle) {
    I = spot_light.intensity / r2; // inside the falloff angle, behaves like a point light
} else if (angle < spot_light.half_coverage_angle) {
    const float top = cos(angle) - cos(spot_light.half_coverage_angle);
    const float bottom = cos(spot_light.half_falloff_angle) - cos(spot_light.half_coverage_angle);
    const float s = powf(top / bottom, 4);
    I = s * (spot_light.intensity / r2);
}
else {
    continue; // no illumination outside the coverage angle
}
```

One mistake that deserves an honorable mention is that I forgot to divide the angles by two, which resulted in twice as powerful spot lights.

## Environment Light

I used TinyEXR to load `.exr` files.

One thing that caused me to waste a lot of time is I did not know that the images had alpha channels. This resulted in the EXR loading function accidentally reading the alpha channel as one of the color channels every three pixels. Here is the result:

![empty_environment_latlong](https://github.com/user-attachments/assets/f3fab366-bb28-4f5e-a2c3-c89dc3cff7b5)

You can make out the 'spherical' shape of the enviroment mapping.

### Random Sampling

I sample cos-weighted with a higher probability of sampling the 'world up' direction. I assume that environment lights are usually brighter towards the sky.

```cpp
const auto e0 = random_uniform_float01();
const auto e1 = random_uniform_float01();
const auto theta = acos(sqrt(e0));
const auto phi = 2 * pi * e1;

const auto x = sin(theta) * cos(phi);
const auto y = cos(theta); // higher probability towards 1
const auto z = sin(phi) * sin(theta);

const auto random_dir = glm::vec3(x, y, z);
const auto inverse_pdf = pi / cos(theta);

const auto I = sample_environment_light(scene.environment_light, random_dir) * inverse_pdf;
```

One mistake here was one mentioned in the lectures, I forgot to multiply by the inverse of the PDF.

16 samples per pixel:
![audi-tt-pisa](https://github.com/user-attachments/assets/04e4d7ef-e4c9-4ba9-a743-6d8489e01acd)

## Tonemapping


## Profiling
![image](https://github.com/user-attachments/assets/3d03b348-bb12-499e-9864-d5f14013be2d)

## Denoising

I wanted to test out image denoising libraries. I chose Intel Open Image Denoiser. 

The implementation was much shorter that I expected.

```cpp
#include <OpenImageDenoise/oidn.hpp>
```

and 40 lines of C++ later I had the denoiser working:

```cpp
...

filter.execute(); // run it

// copy the image back from the denoiser buffer
for (int i = 0; i < width * height; i += 3) {
    auto& current_pixel = image[i];
    current_pixel.r = output[i];
    current_pixel.g = output[i + 1];
    current_pixel.b = output[i + 2];
}
```

scene_pisa.xml, 4 samples per pixel, no denoise:
![audi-tt-pisa4](https://github.com/user-attachments/assets/4815a942-e5e6-4f34-932a-86dcf9615409)

scene_pisa.xml, 4 samples per pixel, denoise:
![audi-tt-pisa4_denoised](https://github.com/user-attachments/assets/8ef8985d-a2e2-495c-af46-db13279c65ae)

The results are disappointing, it seems to rearrange the noise and doesn't reduce it.

**I did a bit of research and this is probably because the denoiser needs additional data like albedo and normal maps to work better. I will work more on this.**


