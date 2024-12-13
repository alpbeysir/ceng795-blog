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

## Preparation for shading

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

The design here is simply to insert a piece of code right before every light in the scene is iterated. The code piece will replace/change the default values obtained from the material. Here is the pseudocode:

```python
do_shading(hit_info, incoming_ray):
    material = correct material from hit_info
    normal, diffuse, specular, ambient = initialize from material

    # this part is the code piece
    for each texture on the object: 
        normal, diffuse, specular, ambient = run texture calculation

    color = ambient * scene ambient color
    color += for each point light, diffuse and specular
    color += for each area light, diffuse and specular

    return color
```




