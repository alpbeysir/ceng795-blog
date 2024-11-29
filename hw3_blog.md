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

