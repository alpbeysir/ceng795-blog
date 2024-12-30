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


```cpp
const auto e0 = random_uniform_float01();
const auto e1 = random_uniform_float01();
const auto theta = acos(sqrt(e0));
const auto phi = 2 * pi * e1;

const auto x = sin(theta) * cos(phi);
const auto y = sin(phi) * sin(theta);
const auto z = cos(theta);

const auto random_dir = glm::vec3(x, y, z);
const auto inverse_pdf = pi / cos(theta);

const auto I = sample_environment_light(scene.environment_light, random_dir) * inverse_pdf;
```



## Profiling
![image](https://github.com/user-attachments/assets/3d03b348-bb12-499e-9864-d5f14013be2d)



