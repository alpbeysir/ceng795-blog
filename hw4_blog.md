# CENG795 Advanced Ray Tracing - Homework 4

The topic of HW4 is various types of texture mapping. As usual, I started building the parsing logic for images and texture maps.

## Parsing & Definitions

I was gradually rewriting the parser and this section will go over the design decisions and general methodology.

### Utility Functions

I have generic functions for reading an XML element with a particular name and type. For example:

```cpp
// This function will try to read a tag named "tagName" with string type that is located inside element "parent"
// If reading fails it will return an optional "defaultValue"
std::string read_string(const tinyxml2::XMLElement* parent, const std::string& tagName, const std::string& defaultValue = "");
```

This function is also defined for ints, floats, vectors etc.
This architecture for parsing the XML if beneficial because sometimes some scenes may lack certain values and the parser needs to set sensible defaults for them.
Additionally, the code is shortened by a significant factor.

### What to store in the Scene struct?

The general flow of the program is that the parser outputs a single Scene struct that contains all the information needed to render the scene. The raytracer is then called with the Scene as a parameter. But with each homework, I noticed that the parser was not only parsing the scene but also doing a lot of pre-processing both to implement performance optimizations and simplify code on the raytracer portion. Yet the unprocessed data was still being stored and used by the code in some places.

I decided to update the parser implementation to change this. For example, the original transformations in the scene file do not need to be stored as the parser calculates composite matrices beforehand. Also, some structures are always used through an 'owner' object such as Image and Mesh. These are transient and don't need to be stored or accessed by the raytracer. 

1. Transformations are read to a temporary place and passed to the part that reads objects. After, they are freed. 
2. Images are passed to the function that reads TextureMaps and the structs are freed. Only the shared image data remains, which can be accessed through the textures that use it.
3. To simplify the raytracer, single triangles are not stored but instead converted to MeshInstance's.
4. Similarly, each mesh is converted to a MeshInstance that includes the BVH of the original mesh. Meshes themselves are not stored.

This refactor was not done to improve performance but to improve readability. Now there are only spheres, mesh instances, textures and triangles.

### Reading Images

```cpp
struct Image {
    std::vector<std::vector<glm::vec3>>* data;
};
```

Images are read using stb_image and placed in a dictionary. This dictionary is passed to the texture reader.

### Reading Texture Maps

Adding support for reading texture maps did not take very long, I spent most of the time deciding the struct layout:

```cpp
struct TextureMap { // definition for texture maps in memory
    TextureType type;
    DecalMode decal_mode;
    union {
        struct {
            std::vector<std::vector<glm::vec3>>* data;
            InterpolationMode interpolation_mode;
        } image;
        struct {
            float bump_factor;
            float noise_scale;
            PerlinNoiseConversion noise_conversion;
            int num_octaves;
        } perlin;
        struct {
            float scale;
            float offset;
            glm::vec3 black_color;
            glm::vec3 white_color;
        } checkerboard;
    };
};
```

I chose to utilize unions, a feature of C++ that I normally don't use often because it may lead to bad memory accesses & hard to read code. But in this case I think it fits perfectly. I observed that texture mapping has a lot of 'decisions' based on enum values and that all of these values are not required simultaneously. I guessed that the texture mapping code that I had not yet written would probably be a huge switch-case statement based on TextureType and other enums inside the union.

After the structure was clearly defined, parsing was a piece of cake thanks to the above-mentioned utility functions:

```cpp
while (element) {
    TextureMap texture_map{};

    auto type = std::string(element->Attribute("type"));
    texture_map.decal_mode = string_to_decal_mode(read_string(element, "DecalMode"));

    if (type == "image") {
        auto image_id = read_int(element, "ImageId", -1);
        texture_map.type = IMAGE;
        texture_map.image.data = images.at(image_id).data;
        // ...
    }
    else if (type == "perlin") {
        texture_map.type = PERLIN;
        texture_map.perlin.noise_conversion = string_to_perlin_noise_conversion(read_string(element, "NoiseConversion"));
        // ...
    }
    else if (type == "checkerboard") {
        texture_map.type = CHECKERBOARD;
        texture_map.checkerboard.offset = read_float(element, "Offset");
        // ...
    }
    else {
        throw std::runtime_error("unknown texture map type");
    }

    scene.texture_maps.push_back(texture_map);
}
```

Basically everything is parsed with the same methodology. The parser code is finally clean.

## Preparation

Before I could perform the texture mapping computations during shading, I realized that I was lacking data at the shading step. I only had the hit point and the normal because that was all I needed before. The shading code did not even know if it was doing a triangle or a sphere. Much refactor needed. 

Instead of passing ad-hoc values to the shading and material/reflections portion of the code, I unified everything under a struct HitInfo. This had to be done eventually:

```cpp
struct HitInfo {
    GeometryType type;
    union {
        const Sphere* sphere;
        struct {
            const Triangle* triangle;
            const SquashedMeshInfo* mesh_info;
        };
    };
    float t_min;
    glm::vec3 point;

    // TODO fix this for spheres
    glm::vec3 local_hit_point;
    glm::vec3 normal;
    int material_id;
    const std::vector<uint>* texture_ids;
};
```

(local_hit_point is broken for spheres. I do not know why, will investigate.)

The geometry/collisions code's job is the generate this struct for a given Ray. 


## The actual work

The design here is simply to insert a piece of code right before every light in the scene is iterated. The code piece will replace/change the default values obtained from the material using the texture data. Here is the pseudocode of the shading logic and the inserted code:

```python
do_shading(hit_info, incoming_ray):
    material = correct material from hit_info
    normal, diffuse, specular, ambient = initialize from material and object

    # this part is the inserted code
    for each texture on the object: 
        normal, diffuse, specular, ambient = run texture calculation

    color = ambient * scene ambient color
    color += for each point light, diffuse and specular
    color += for each area light, diffuse and specular

    return color
```

The critical code that calculates the new shading values:

```cpp
const auto uv = get_uv(scene, hit_info);
for (int i = 0; i < hit_info.texture_ids->size(); i++) { // for each texture on the object
    auto texture_id = hit_info.texture_ids->at(i);
    switch (const auto& texture_map = scene.texture_maps[texture_id]; texture_map.decal_mode) { // look at the texture decal mode to decide what to replace/blend
        case REPLACE_KD:
            diffuse = sample_texture(texture_map, uv, hit_point);
            break;
        case BLEND_KD:
            diffuse = (diffuse + sample_texture(texture_map, uv, hit_point)) / 2.0f;
            break;
        case REPLACE_KS:
            specular = sample_texture(texture_map, uv, hit_point);
            break;
        case REPLACE_NORMAL: {
            // to be discussed later
        }
        case BUMP_NORMAL:
            // bump mapping did not make the deadline :(
            break;
        case REPLACE_ALL: // replace all components,
            auto sample = sample_texture(texture_map, uv, hit_point);
            diffuse = sample;
            specular = sample;
            ambient = sample;
            break;
}
```

Most of the texture types are straightforward and involve one or two computations. The only complicated one is REPLACE_NORMAL which implements normal mapping. I will describe my struggles with that later.

The code depends on two external functions that perform:

1. sample_texture: nearest, bilinear, perlin, checkerboard sampling.
2. get_uv: uv for sphere and triangle.

### sample_texture (Texture Sampling)

Some portions are omitted for simplicity.

```cpp
glm::vec3 sample_texture(const TextureMap& texture_map, const glm::vec2& uv, const glm::vec3& position) {
    switch (texture_map.type) { // look at the texture map type
        case IMAGE: {
            const auto& image_data = *texture_map.image.data; // copy mistake (&)
            const float pixel_x = uv.x * width;
            const float pixel_y = uv.y * height;
            switch (texture_map.image.interpolation_mode) {
                case NEAREST: {
                    // return one sample from the image
                }
                case BILINEAR: {
                    // return four averaged samples from the image
                }
                case TRILINEAR: {
                    throw std::runtime_error("optional");
                }
            }
        }
        case PERLIN: {
            const glm::vec3 scaled_position = position * texture_map.perlin.noise_scale;
            float noise_value = 0.0f;
            float amplitude = 1.0f;
            float frequency = 1.0f;
            float total_amplitude = 0.0f;

            for (int i = 0; i < texture_map.perlin.num_octaves; ++i) {
                noise_value += amplitude * perlin_noise(scaled_position * frequency);
                total_amplitude += amplitude;
                amplitude *= 0.5f;
                frequency *= 2.0f;
            }
            noise_value /= total_amplitude;

            noise_value = apply_perlin_conversion(noise_value, texture_map.perlin.noise_conversion); // don't forget to map the noise value!
            return glm::vec3(noise_value) * texture_map.perlin.bump_factor;
        }
        case CHECKERBOARD: { 
            // exactly the same as the PDF code
        }
    }
}
```

**Stupid mistakes**

1. Initially for each call of sample_texture for an image texture I was copying the image data before accessing it. This obviously resulted in horrible performance. A short profiling session fixed it.
2. For the bilinear implementation I forgot to add the bottom 2 pixels to the return value. For a while all bilinear textures looked smeared horizontally:
![galactica_static_1](https://github.com/user-attachments/assets/8b93b4da-41c7-4bf3-8d20-43299879ab67)
 

**Checkerboard?**

For the CHECKERBOARD texture type, I could not find any usages. All scenes use an image of a checkerboard. I probably missed something.

**Perlin noise misunderstanding**

Initially I thought that I was supposed to implement 2D perlin and sample it using UV coordinates. After doing this and seeing that the noise in the example outputs is 3D, I rewrote the perlin_noise function to take in a 3D vector and passed the hit_point to it.

### get_uv (UV Calculation)

This function is pretty straightforward but I wanted to make some comments about the issues I encountered: 
(I omitted some parts for brevity)

```cpp
glm::vec2 get_uv(const Scene& scene, const HitInfo& hit_info) {
    switch (hit_info.type) {
        case GeometryType::Sphere: {
            // for some reason hit_info.local_hit_point outputs literal garbage
            // as a band-aid I recalculate it based on the global hit_point here
            auto local_hit_point = hit_info.point - hit_info.sphere->position;
            local_hit_point /= hit_info.sphere->radius;
 
            // (same as slides)

            return {u, v};
        }
        case GeometryType::Triangle: {
            // ...

            const glm::vec3 edge0 = triangle.edge0;
            const glm::vec3 edge1 = triangle.edge1;
            const glm::vec3 edge2 = local_hit_point - v0;

            const float d00 = dot(edge0, edge0); // can be precomputed
            const float d01 = dot(edge0, edge1); // can be precomputed
            const float d11 = dot(edge1, edge1); // can be precomputed
            const float d20 = dot(edge2, edge0);
            const float d21 = dot(edge2, edge1);

            const float denom = d00 * d11 - d01 * d01; // can be precomputed
            const float v = (d11 * d20 - d01 * d21) / denom;
            const float w = (d00 * d21 - d01 * d20) / denom;
            const float u = 1.0f - v - w;

            return u * uv0 + v * uv1 + w * uv2;
        }
    }
}
```

**Remarks**

The recalculation of local_hit_point for spheres is not good for performance. I will fix this.

Additionally, a portion of the code for triangles can be precalculated because they do not depend on the hit point.
- d00, d01, d11, denom

### Normal Mapping  

Here I wonder if instead of transforming everything about the object to world space I should have transformed the light stuff to object space. Perhaps I can save on the last matrix multiplication.

For spheres, tangent space is calculation on-the-fly as it depends on the exact hit point. This is because a sphere is not flat.

```cpp
const glm::vec3 normal_sample = sample_texture(texture_map, uv, hit_point) * 2.0f - glm::vec3(1.0f, 1.0f, 1.0f); // stupid mistake
case GeometryType::Sphere: {
        // calculate TBN matrix
        auto local_hit_point = hit_info.point - hit_info.sphere->position;
        local_hit_point /= hit_info.sphere->radius;
        const auto t = glm::vec3(2 * pi * local_hit_point.z, 0, -2 * pi * local_hit_point.x);
        const auto local_norm = (hit_info.local_hit_point - hit_info.sphere->position) / hit_info.sphere->radius;
        const auto b = cross(t, local_norm); // instead of finding B similar to T, I used the original normal
        const auto tbn = glm::mat3(t, b, local_norm);

        // transform the sampled normal
        const auto calculated_local_norm = tbn * normal_sample;
        norm = /* transformed calculated_local_norm */;
        break;
    }
    case GeometryType::Triangle: {
        // pretty straightforward because TBN is precomputed
        const auto calculated_local_norm = normalize(hit_info.triangle->tbn * normal_sample);
        norm = /* transformed calculated_local_norm */;
        break;
    }
```

**Stupid mistakes**

1. Initially instead of subtracting a vector of ones from the scaled normal texture sample I was adding it. This resulted in normals ranging from [1, 3] which resulted in:



## Gallery




