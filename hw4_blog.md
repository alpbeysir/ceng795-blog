# CENG795 Advanced Ray Tracing - Homework 4

The topic of HW4 is various types of texture mapping. As usual, I started building the parsing logic for images and texture maps.

## Parsing

By this point my parser is well-structured so it is easy to add support for new tags. I have generic functions for reading an XML element with a particular name and type. For example:

```cpp
// This function will try to read a tag named "tagName" with string type that is located inside element "parent"
// If reading fails it will return an optional "defaultValue"
std::string read_string(const tinyxml2::XMLElement* parent, const std::string& tagName, const std::string& defaultValue = "");
```

This architecture for parsing the XML if beneficial because sometimes some scenes may lack certain values and the parser needs to set sensible defaults for them.
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



